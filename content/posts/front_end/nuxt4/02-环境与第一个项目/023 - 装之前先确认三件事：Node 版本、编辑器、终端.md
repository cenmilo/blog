---
url: /posts/aa4705a26ec82e0423a173f3ee1b1574/
title: 023 - 装之前先确认三件事：Node 版本、编辑器、终端
date: 2026-10-19T11:00:00+08:00
updated: 2026-10-19T11:00:00+08:00
author: cmdragon

summary:
  摘要：官方在 Installation 这一节把"开始之前需要什么"缩成了三项：Node.js 22.x 或更新版本（推荐用 active LTS）、一个文本编辑器（没有强制要求的 IDE，但推荐 VS Code 加官方 Vue 扩展，也就是原来的 Volar，或者 WebStorm 等 JetBrains 系 IDE；用 Neovim 这类编辑器要自己按 Vue Language Tools 的指南配 Vue 语言服务器）、一个终端（用来跑 Nuxt 命令）。另外还附了四条"最优配置"的补充说明：Node 要用偶数版本（22、24 这类）；Neovim 配 Vue TypeScript 插件时 `location` 要指向 `@vue/language-server` 的包目录而不是它的二进制；Windows 上 HMR 很慢可以试试 WSL；Windows 本地开发别用 `localhost:3000`，改用 `127.0.0.1` 打开会快很多。这一节还留了一条退路：暂时不想装任何环境，可以直接用官方给的在线沙箱。

categories:
  - 前端开发

tags:
  - nuxt4
  - 环境准备
  - node
  - 编辑器
  - 入门

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

前面二十来篇都在讲是什么、为什么。从这一篇开始，我们要真的把一个项目跑起来。动手之前先花一篇把门槛确认清楚。官方在 Installation 页开头列的三项前置，很多人扫一眼就跳过了，但它附的那四条补充说明里，有两条能直接省掉你之后的折腾。

第一项，Node.js。官方的原话是，Node.js，22.x 或更新，但我们推荐用 active LTS 版本。

这里其实是三个信息叠在一起。

下限是 22.x，比这个旧的版本，官方没有承诺能跑。

推荐 active LTS，而不是最新的。active LTS 指 Node 官方发布时间表里正处于活跃维护期的那个大版本，具体是哪个要看那张表。

要用偶数版本。补充说明里写得很直接，Node.js，确保使用偶数编号的版本，22、24 等。

官方在这里只给了结论没写理由，理由在 Node 自己的发布制度里，这是我的推断，不是文档内容。你只要记住，看版本号的个位数是不是偶数，别纠结发行版名字。

查自己现在是什么版本、以及怎么切版本，属于 Node 或版本管理工具的事，Nuxt 文档没管。确认完了再往下。

第二项，编辑器。官方先给了一个很宽松的话，文本编辑器，没有 IDE 要求，但我们推荐 Visual Studio Code 加官方 Vue 扩展，之前叫 Volar，或者 WebStorm。

然后依次给了三类选择。

VS Code 加官方 Vue 扩展，`Vue.volar`。注意官方那句以前叫 Volar，你在市场上搜 Volar 也能搜到，它是同一个扩展改了名。

WebStorm 以及其它 JetBrains IDE。官方的说法是这些开箱即用地提供了很好的 Nuxt 支持，也就是不用你额外配插件。

其它编辑器，例如 Neovim。官方要求你自己去配 Vue Language Server 支持，并指了 Vue Language Tools 的设置指南。

没有 IDE 要求这句是真的没有，跑 Nuxt 靠的是终端里的命令，编辑器只影响你写代码时的提示好不好用。

第三项，终端。官方原话是，终端，用来跑 Nuxt 命令。

只有一条要求。官方还顺手补了一句，如果你用 VS Code，可以直接开它的集成终端，不必另开一个窗口。

那四条补充说明里，两条要单独记住。

一，Neovim 用户，`location` 指向包目录，不是二进制。官方原话，Neovim，配置 Vue TypeScript 插件时，确保 `location` 指向 `@vue/language-server` 包的目录，而不是它的二进制文件。官方在这条后面附了一份能正常工作的配置的 Neovim 设置指南链接。

官方特意说指南里有一份能工作的配置，说明这是一个真的会踩的坑。用 Neovim 的话去照那份抄，别自己拼路径。

二，Windows 用户有两条独立的建议。第一条，WSL，如果你在 Windows 上遇到 HMR 很慢，可以试试用 WSL，也就是 Windows Subsystem for Linux，这可能会解决一些性能问题。

第二条，Windows 慢 DNS 解析，在 Windows 上访问本地开发服务器时，不要再用 `localhost:3000`，改用 `127.0.0.1`，浏览器里的加载体验会快很多。

第二条几乎零成本，改一下你地址栏里输入的域名。它讲的不是 Nuxt 慢，是浏览器把 `localhost` 这个名字解析成 IP 的那一下慢。第一条要不要动取决于你的体感，官方用的措辞是 may solve，可能会解决，不是保证。

还不想装环境，官方给了退路。Installation 页第一小节不是安装步骤，是一个在线沙箱。如果你只是想在浏览器里玩玩 Nuxt、不想配置项目，可以用这个在线沙箱，Open on StackBlitz，带代码编辑器和终端访问的在线 playground。

链接是 `https://stackblitz.com/github/nuxt/starter?file=app%2Fapp.vue`，直接打开官方 starter 仓库、并把 `app/app.vue` 这个文件给你摊开。先看看长什么样再装，是完全合理的第一步，等你确认要动手了，再回到上面那三项前置。

## 自查

- [ ] 你的 Node 版本是 22.x 或更新的**偶数**版本吗？
- [ ] 你用的编辑器属于官方列的哪一类：VS Code（要装 Vue 扩展）、JetBrains 系（不用装）、还是其它（要自己配语言服务器）？
- [ ] 用 Neovim 的话，`location` 指的是包目录还是二进制？
- [ ] Windows 用户：地址栏输入的是 `localhost:3000` 还是 `127.0.0.1`？HMR 慢不慢，要不要试 WSL？
- [ ] 只是想围观的话，你知道可以先用 StackBlitz 沙箱，不用装任何东西吗？

## 下一篇

024 开始装：一条 `create` 命令建项目，一条 `dev` 命令起服务，中间那个 `-o` 到底写了什么、`package.json` 里那五个脚本各自什么时候跑。
