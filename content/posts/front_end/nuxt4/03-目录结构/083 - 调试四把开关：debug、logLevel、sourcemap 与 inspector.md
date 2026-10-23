---
url: /posts/eca694ab81a4c778e97f7a8391d0b755/
title: 083 - 调试四把开关：debug、logLevel、sourcemap 与 inspector
date: 2026-11-08T11:00:00+08:00
updated: 2026-11-08T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方把"看不清 Nuxt 内部发生了什么"这件事拆成四把互相独立的开关。debug 设为 true 打开调试模式，官方对它的描述是"目前它会打印服务端的钩子名与耗时，并在浏览器里也打印钩子的参数"，类型是 boolean、默认 false，还可以设成一个对象来只开指定选项。logLevel 是构建日志的级别，类型 string、默认 "info"，官方特别写明"在 CI 里或没有 TTY 时默认降为 'silent'，并且这个值在 Vite 里被当作 'silent' 使用、在 webpack 里被当作 'none'"——这就是 CI 日志突然安静的原因。sourcemap 默认是 { server: true, client: false }，可以按 server 与 client 分别设 true、false 或 'hidden'，官方对 'hidden' 的定义是"生成 sourcemaps 但不在最终产物里包含引用"；而 Debugging 页的说法是 sourcemap 默认对服务端构建开启、对客户端在 dev 模式开启。第三把是命令行上的 --inspect，Debugging 页与 dev 命令页都给了一条件重要限制：Node.js 与 Chrome 进程必须运行在同一平台上，在 Docker 里不生效。第四把是 IDE 配置，官方给了 VS Code 与 JetBrains 两份，共同点是两条配置（client: chrome 指向 http://localhost:3000 与 server: nuxt 指向 node_modules/nuxt/bin/nuxt.mjs 传 dev 参数）再加一条 compound 把它们合起来。

categories:
  - 前端开发

tags:
  - nuxt4
  - 调试
  - sourcemap
  - 日志
  - inspector

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

050-089 收尾这四篇讲生成物。这一篇先讲怎么看清它们，官方把调试拆成四把互相独立的开关，用错一把就会白等一次构建。

第一把，`debug`，看钩子。官方原话，设为 `true` 以启用调试模式。目前，它会打印服务端的钩子名与耗时，并在浏览器里也打印钩子的参数。你也可以把它设成一个对象，以启用特定的调试选项。类型 boolean，默认值 false。

两个词要抓住。

目前，官方原文 At the moment，这句是在说输出内容会变，别把它当稳定接口。

钩子名与耗时，这条覆盖的正是 019 那条生命周期链，你想知道哪一步慢，它给的是钩子级数据。这与 043 的 `nuxt analyze`，量产物、037 的 `--profile`，量构建，是三个不同工具。

第二把，`logLevel`，决定你看不看得见。官方原话，构建日志的级别。在 CI 里或当一个 TTY 不可用时，默认为 `silent`，并且这个值在 Vite 里被用作 `silent`、在 webpack 里被用作 `none`。类型 string，默认值 `info`。

这一条解释了一个非常具体的现象，本地有日志，CI 上什么都没有。官方给的答案是自动降级。

第二半句还带出一条 builder 相关的差别，088 会讲构建器可切换，同一个 Nuxt 配置值，会被翻译成不同构建器各自的取值。

第三把，`sourcemap`，决定报错能不能定位到你写的代码。官方原话，配置是否为服务端和或客户端包生成 sourcemaps，以及如何生成。如果设为一个单独的布尔值，该值同时应用于 server 与 client。此外，`hidden` 选项对 server 与 client 都可用。

| 取值 | 官方说明 |
| --- | --- |
| `true` | 生成 sourcemaps 并在最终产物里包含 source 引用 |
| `false` | 不生成任何 sourcemap |
| `hidden` | 生成 sourcemaps，但不在最终产物里包含引用 |

```json
{
  "server": true,
  "client": false
}
```

上面这份 JSON 是官方给的默认值。

Debugging 页那句与它对照着读。Sourcemaps 默认对你的服务端构建开启，对客户端在 dev 模式下开启，但你可以在配置里更具体地启用它们。

两者一致，默认是服务端有、客户端生产没有。于是这条排查口诀成立，生产环境浏览器里那份堆栈看不懂，第一反应是 `sourcemap.client` 是 `false`。

`hidden` 这一档存在的意义值得说清，官方只给了定义，这一句是我从生成但不带引用推出的用途，产物里不带引用，就不会在浏览器控制台出现 sourcemap 提示，但你手里仍然有 sourcemap 文件可以事后还原堆栈，要留给排查用、又不想暴露给用户的，就是这一档。

第四把，命令行 `--inspect`。Debugging 页给的命令与解释。

官方原话，你可以使用 Node inspector 来调试 Nuxt 的服务端。

```bash
nuxt dev --inspect
```

这会在调试器激活的状态下以 `dev` 模式启动 Nuxt。如果一切正常，你的 Chrome DevTools 里会出现一个 Node.js 图标，你可以附着到调试器上。

034 抄过 dev 页那句更精确的版本，默认端口 `127.0.0.1:9229`、服务端代码跑在 Nitro 工作线程里、CLI 进程自己占下一个端口 9230。那一节和这一篇是一对，这里补上官方那条限制。

官方用 important 写着，请注意，Node.js 与 Chrome 进程需要运行在同一平台上。这在 Docker 里面不起作用。

这一条对用 032 那份 dev container 的人很重要，容器里跑 Nuxt、宿主机开 Chrome，正是跨平台的组合。官方在这里没给绕法，我就不编绕法，该走的是另外两条能用的路，IDE 配置，下一节，或 `nuxt dev --inspect` 后由编辑器连端口。

第五个入口，IDE 里调试，官方给两份配置。

VS Code。官方给的这份配置里有两个注释值得看。

```json5
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "chrome",
      "request": "launch",
      "name": "client: chrome",
      "url": "http://localhost:3000",
      // this should point to your Nuxt `srcDir`, which is `app` by default
      "webRoot": "${workspaceFolder}/app"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "server: nuxt",
      "outputCapture": "std",
      "program": "${workspaceFolder}/node_modules/nuxt/bin/nuxt.mjs",
      "args": [
        "dev"
      ],
    }
  ],
  "compounds": [
    {
      "name": "fullstack: nuxt",
      "configurations": [
        "server: nuxt",
        "client: chrome"
      ]
    }
  ]
}
```

三处要点。

`webRoot` 那句注释直接连着 051、079 的知识，它必须指向你的 `srcDir`，而 `srcDir` 默认是 `app`。改了 `srcDir` 的人，这份配置要跟着改。

`server` 那条的 `program` 是 `node_modules/nuxt/bin/nuxt.mjs` 加 `args: ["dev"]`，它就是 024 那张脚本表底下那层，`npm run dev` 到 `nuxt dev`。

`compounds` 才是这份配置的价值，`fullstack: nuxt` 把服务端 Node 与浏览器 Chrome 两个会话同时起来，同构应用里同一个请求两侧各跑一次，013，两侧要能同时下断点。

官方还留了一条偏好项，如果你更喜欢你平常用的浏览器扩展，在上面 chrome 配置里加上这一行 `"userDataDir": false`。

JetBrains 那份是同构的三份配置，官方让你在项目根建一个 `nuxt.run.xml` 并粘进去，里面同样是 `client: chrome`，一个 `JavascriptDebugType`、`uri="http://localhost:3000"`，`server: nuxt`，一个 `NodeJSConfigurationType`、`application-parameters="dev"`、`path-to-js-file="$PROJECT_DIR$/node_modules/nuxt/bin/nuxt.mjs"`，与 `fullstack: nuxt`，`CompoundRunConfigurationType`。

以及最后一条，其它 IDE 这一节官方没有配置，只写了如果你有别的 IDE 并愿意贡献示例配置，欢迎开 PR。

一次排错该按什么顺序开，我的用法。这一节是我的操作建议，官方没有给顺序。

先 `logLevel`，确认日志级别没被降成 `silent`，034 的 `--logLevel` 就能临时改。

再 `sourcemap`，看不懂堆栈就确认对应那一侧有没有 sourcemap。

想知道哪一步慢，`debug: true` 看钩子耗时，服务端。

要在代码里下断点，`--inspect`，非 Docker，或那份 IDE compound。

要看框架替我生成了什么文件，052 的 DevTools Virtual Files，要请求接口，041 的 `nuxt curl`。

## 自查

- [ ] `debug: true` 具体打印什么？官方哪两个字在提醒你别依赖它？
- [ ] 为什么 CI 上什么都没打印？那个降级值在 Vite 与 webpack 里分别被翻译成什么？
- [ ] 生产浏览器堆栈不可读，默认值是哪一项造成的？
- [ ] `hidden` 与 `true` 的差别是什么？什么时候选它？
- [ ] 你在容器里跑 Nuxt、宿主机开 Chrome，`--inspect` 为什么不生效？能改用哪两条路？
- [ ] 改了 `srcDir` 之后，那份 VS Code 配置里哪个字段必须跟着改？

## 下一篇

084 讲 `watch` 与 `watchers`：改了文件不热重载时，先看 `watch` 里字符串与正则各自的基准（都是 `srcDir`，与 079 那批配置项不同），再看 `chokidar.ignorePermissionErrors` 默认为 true 会在容器里静默吞掉什么。
