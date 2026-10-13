---
url: /posts/31d6305d1d8f9b537a19ef3914dc4b55/
title: 699 - 模块开发排错：sourcemaps、--inspect 与 IDE 配置能查到哪一步
date: 2027-06-02T09:00:00+08:00
updated: 2027-06-02T09:00:00+08:00
author: cmdragon

summary:
  摘要：调试那一节只给了四样东西：sourcemaps 的默认状态（**服务端构建默认开、客户端只在开发模式开**）、`nuxt dev --inspect`、一份 VS Code 配置、一份 JetBrains 配置。它对模块作者最有用的是那份 VS Code 配置里的第二项——`program` 指向 `node_modules/nuxt/bin/nuxt.mjs`、`args` 是 `["dev"]`，也就是说它启动的正是跑着你模块 `setup` 的那个进程。而 `--inspect` 有一条硬限制：Node.js 与 Chrome 必须在同一平台，**Docker 里不行**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 调试
  - sourcemap
  - IDE 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先分清你在调试哪一段

一个模块的错可能落在三个地方，各自的可观测手段完全不同：

- 浏览器里（组件没渲染、组合式函数拿不到值）
- 服务端运行时（接口返回不对、SSR 报错）
- 构建期的模块进程（`setup` 没生效、注册的资产不见了、类型没生成）

调试那一节写的是前两段的手段，但第三段其实也顺带覆盖了——原因在下面那份 IDE 配置里。

## sourcemaps：默认值分两侧

原文那句是把三个状态一次说清的：

> sourcemaps 对服务端构建默认启用，对客户端构建则在开发模式启用；你可以在配置里更精确地开启。

配置形状有两种：

```ts
export default defineNuxtConfig({
  // or sourcemap: true
  sourcemap: {
    server: true,
    client: true,
  },
})
```

这一条对排错的直接影响很实际：**如果你在生产构建里看客户端栈，默认是映射不上的**，要自己把 `client` 打开。

## `--inspect`：能attach，但有一条平台限制

服务端可以用 Node inspector 调试，命令是一行：

```bash
nuxt dev --inspect
```

文档说明它的作用：这会以调试器激活的状态启动 Nuxt 的开发模式；一切正常的话，**Chrome DevTools 上会出现一个 Node.js 图标，你可以附加过去**。紧跟着一条重要提示划定了边界：

> Node.js 与 Chrome 这两个进程必须运行在同一平台。在 Docker 里面这不工作。

也就是说容器里跑 dev、宿主机上开浏览器调试，这条路是不通的。文档**没有给出容器场景的替代方案**，也没说 `nuxt build` 侧有没有对应的 inspect 行为。

## VS Code 那份配置里，三项各管一段

文档给的配置有三个 `configurations`/`compounds` 条目，注释也保留了原意：

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

三处细节各自有用：

- `webRoot` 那行注释是文档自己写的：**它应该指向你的 `srcDir`，而默认值是 `app`**。改了 `srcDir` 却没有改这一项，客户端断点会对不上源文件。
- `server: nuxt` 这一项的 `program` 直接指向 `node_modules/nuxt/bin/nuxt.mjs`，`args` 就是 `["dev"]`。**这就是那个跑模块 `setup` 的进程**：你的模块代码、`nuxt.hook` 的回调、Kit 调用，全都在这一个 Node 进程里执行。想调试模块本身，就用这一项，把断点下在模块的源码而不是 runtime 文件里。
- `outputCapture: "std"` 把标准输出收进调试面板——模块用 `useLogger` 打的东西会出现在这里。

`compounds` 那个 `fullstack: nuxt` 是把上面两项一起启动。文档还补了一句：如果你希望用自己平时装的浏览器扩展，就在 chrome 那项里加 `"userDataDir": false`。

## JetBrains 那份是两步

官方给的名字是 `nuxt.run.xml`，位置在项目根目录，做法是新建文件再粘进去。三段分别对应：`client: chrome` 用 `JavascriptDebugType`、`uri` 是 `http://localhost:3000`、带 `useFirstLineBreakpoints="true"`；`server: nuxt` 用 `NodeJSConfigurationType`、`application-parameters="dev"`、`path-to-js-file` 指向 `$PROJECT_DIR$/node_modules/nuxt/bin/nuxt.mjs`；`fullstack: nuxt` 用 `CompoundRunConfigurationType` 把前两个 `toRun` 起来。

页面最后一节是 Other IDEs：如果你用别的 IDE 并愿意贡献示例配置，可以开一个 PR。**也就是说 4.x 的这一节只有 VS Code 与 JetBrains 两份现成配置。**

## 模块专属的排错手段，其实在钩子和日志里

调试页讲的是通用手段。模块的问题更多是"什么时候轮到我执行"，这就要看钩子那张表：

| 想知道的事 | 看哪个钩子 |
| --- | --- |
| 我的模块 setup 跑了多久 | `module:done`（构建期那张表里**唯一在描述中带上耗时**的一行） |
| 模块整批是否都注册完了 | `modules:before` / `modules:done`（注意与单数形式差一个 `s`） |
| 虚拟应用生成到哪一步 | `app:resolve`、`app:templates`、`app:templatesGenerated`、`builder:generateApp` |
| 类型为什么没生成 | `prepare:types`（描述写明它在 `@nuxt/cli` 写出 tsconfig 与 `nuxt.d.ts` **之前**） |
| 构建为什么失败 | `build:error` |
| 开发期文件事件有没有传进来 | `builder:watch`（唯一把"只在开发期"写进描述的一行） |
| 接口的响应类型从哪来 | `server:routes`（用途是给 `$fetch`/`useFetch` 推类型） |

配合输出侧：`useLogger('my-module', { level: options.quiet ? 0 : 3 })` 是官方示例里的写法，把噪声收在一个模块选项上；需要用户决策或看进度时才动 `useTerminal`，而且在没有交互宿主时它会退回成打日志。

还有一条不需要调试器就能发现的线索：**setup 超过 1 秒，Nuxt 自己会警告**。这一条通常比任何断点都先告诉你"逻辑放错地方了"。

## 五步顺序

1. 先定层：浏览器、服务端运行时，还是构建期进程。
2. 构建期的问题用 `server: nuxt` 那项配置启动，断点下在模块源码；容器场景走不通就换到宿主机。
3. 需要看客户端源码映射时确认 `webRoot` 指向你的 `srcDir`，生产客户端栈再单独开 `sourcemap.client`。
4. 怀疑"没执行"就查时机：把观察点放进对应钩子，`module:done` 那份耗时是现成的。
5. 最后才怀疑注册没生效：Kit 侧的名字、路径 resolver、以及有没有把 `@nuxt/kit` 放进 `dependencies`。

到这里，模块与扩展开发这一段的正文全部结束。下一篇进入实验特性：那 57 个开关的默认值、"提前启用"到底意味着什么，以及官方对风险的声明。

## 下一篇

下一篇 700：实验特性机制：默认开、默认关与那句“可能被移除或修改”

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/debugging
- https://nuxt.com/docs/4.x/api/advanced/hooks
- https://nuxt.com/docs/4.x/api/kit/logging
