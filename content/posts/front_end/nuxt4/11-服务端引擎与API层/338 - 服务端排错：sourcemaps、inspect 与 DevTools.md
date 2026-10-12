---
url: /posts/87ba115df65308e6ada38d24d45b6494/
title: 服务端排错：sourcemaps、inspect 与 DevTools
date: 2027-02-01T11:00:00+08:00
updated: 2027-02-01T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：服务端代码报错比前端更难查，因为不在你熟的那套浏览器工具里。这一篇把官方给的几条服务端排错手段并起来：sourcemaps（服务端构建默认开、客户端 dev 模式开）、`nuxt dev --inspect` 起 Node 调试器接 Chrome DevTools（Node 与 Chrome 要同平台、Docker 里不行）、`debug: true` 打印钩子名与耗时（服务端）和钩子参数（浏览器），`nuxt devtools` 按项目开/关 DevTools，以及 VS Code / JetBrains 的调试配置（webRoot 要指 srcDir，默认 app）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 调试
  - sourcemap
  - inspect
  - DevTools
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`server/api/*.ts` 里抛个错，你在浏览器控制台常常看不到清楚的栈——因为它在服务端跑。这一篇把官方给的服务端（和全栈）调试手段收齐。

## sourcemaps：服务端默认就有

官方 debugging 页第一句：

> Sourcemaps are enabled for your server build by default, and for the client build in dev mode, but you can enable them more specifically in your configuration.

也就是**服务端构建默认开 sourcemap**（栈能对上源码），客户端只在 dev 开。要更细：

```ts
export default defineNuxtConfig({
  // or sourcemap: true
  sourcemap: {
    server: true,
    client: true,
  },
})
```

这跟 nuxt-config 的 `sourcemap` 默认值（server:true / client:false，见 083）一致。`'hidden'` 选项是"生成但产物里不放引用"。

## nuxt dev --inspect：接 Chrome DevTools 调服务端

> You can use Node inspector to debug Nuxt server-side.

```bash
nuxt dev --inspect
```

> This will start Nuxt in `dev` mode with debugger active. If everything is working correctly a Node.js icon will appear on your Chrome DevTools and you can attach to the debugger.

一条必须知道的限制（important）：

> Note that the Node.js and Chrome processes need to be run on the same platform. This doesn't work inside of Docker.

Node 和 Chrome 要在**同一平台**，**Docker 里用不了** `--inspect`。这解释了为什么容器里调试要另走 IDE attach 那套。

## debug: true：打印钩子名和耗时

nuxt-config 的 `debug` 选项（默认 false）：

> Set to `true` to enable debug mode. At the moment, it prints out hook names and timings on the server, and logs hook arguments as well in the browser. You can also set this to an object to enable specific debug options.

`debug: true` 让**服务端打印钩子名与耗时、浏览器额外打印钩子参数**——排查"哪个钩子/什么时候跑的"很有用。官方这句用了 "At the moment"，也就是这套打印行为是当前实现、未来可能变，别把它当稳定契约依赖。

## DevTools 按项目开关

`nuxt devtools` 命令（api/commands/devtools 页）：

> Running `nuxt devtools enable` will install the Nuxt DevTools globally, and also enable it within the particular project you are using. It is saved as a preference in your user-level `.nuxtrc`. If you want to remove devtools support for a particular project, you can run `nuxt devtools disable`.

即 `nuxt devtools enable` 全局装 + 本项目的开启，偏好写在用户级 `.nuxtrc`（074 提过 `.nuxtrc` 那层）；`disable` 关某个项目。命令形状 `nuxt devtools <COMMAND>`，COMMAND 是 `enable|disable`。DevTools 里能看 payload、hooks（284 提过 Hooks 面板看事件）等。

## IDE 断点：VS Code 与 JetBrains

官方给了两套配置骨架。

**VS Code**（launch 配置）关键一条注释：

```json5
// this should point to your Nuxt `srcDir`, which is `app` by default
"webRoot": "${workspaceFolder}/app"
```

**`webRoot` 要指到你的 `srcDir`**（默认 `app`）——指错了断点会命不中源码。配置里给的是两条：`client: chrome`（浏览器侧）+ `server: nuxt`（Node，program 指到 `node_modules/nuxt/bin/nuxt.mjs`、args `["dev"]`），外加一个 `fullstack` compound 一起起。

**JetBrains**（IntelliJ/WebStorm/PhpStorm）：在项目根建 `nuxt.run.xml`，粘官方给的那段 client/server/fullstack 三配置。

（官方还注明：偏好浏览器扩展的话，在 chrome 配置里加 `"userDataDir": false`；用别的 IDE 欢迎贡献配置示例。）

## 排查路径建议

服务端报错，按这条顺序：

1. 先确认 `sourcemap`（服务端默认开，栈能对上源码）。
2. 想知道"哪个钩子在何时跑了什么参数" → `debug: true`。
3. 要打断点单步 → `nuxt dev --inspect` + Chrome（非 Docker），或 IDE（VS Code `webRoot` 指对 srcDir）。
4. 想看 payload/hook/路由可视化 → DevTools（`nuxt devtools enable`）。

## 自查

- 服务端构建默认有 sourcemap，栈通常能对上源码；客户端 sourcemap 只在 dev。
- `nuxt dev --inspect` 接 Chrome 有"同平台、Docker 里不行"的限制；容器内改用 IDE attach。
- `debug: true` 打印钩子名/耗时（服务端）与参数（浏览器），官方标"At the moment"、别当稳定 API。
- VS Code `webRoot` 一定要指 `srcDir`（默认 `app`），否则断点命不中。

## 下一篇

下一篇 339 给服务端这层收尾：一张 Nitro 能力地图，把 P11 讲过的路由、中间件、插件、runtimeConfig、routeRules、预渲染、部署串成索引。

## 本文依据

- Debugging（`guide/going-further/debugging`）：Sourcemaps（"enabled for your server build by default, and for the client build in dev mode"、`sourcemap:{server,client}` 示例）；Node Inspector（`nuxt dev --inspect`、Chrome DevTools Node.js 图标、important "Node.js and Chrome processes need to be run on the same platform. This doesn't work inside of Docker."）；VS Code 配置（webRoot 注释 "should point to your Nuxt srcDir, which is app by default"、client/server/fullstack 三 config、userDataDir:false）；JetBrains nuxt.run.xml 三配置。
- nuxt-config debug 节（`api/nuxt-config`）："Set to true ... At the moment, it prints out hook names and timings on the server, and logs hook arguments as well in the browser."，默认 false；sourcemap 默认（见 083）。
- nuxt devtools（`api/commands/devtools`）：`nuxt devtools enable/disable`、全局安装 + 写到用户级 `.nuxtrc`、COMMAND=enable|disable。
