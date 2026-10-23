---
url: /posts/914e2e4bc0b4cf712bd3d0855a47bcde/
title: 360 - hooks 节：在 nuxt.config 里直接写构建钩子
date: 2027-02-09T09:00:00+08:00
updated: 2027-02-09T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：构建期钩子（297 讲的两个注册位置之一）就在 nuxt.config 的 hooks 节里写。nuxt-config 说这些钩子"通常是模块在用，但 nuxt.config 里也可用"；内部钩子名用冒号分段（build:done），但为方便起见 nuxt.config 里可以写成层次对象——把 build:done 拆成 hooks: { build: { done(){} } }。这一篇讲这个写法、它和模块里 nuxt.hook 的关系（订的是同一批 Nuxt Hooks (Build Time)），以及层次对象只是"书写糖"、本质还是冒号名。给一个在 build:done 里往 buildDir 写额外文件的官方示例。

categories:
  - 前端开发

tags:
  - nuxt4
  - hooks节
  - 构建钩子
  - build-done
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


297 讲"构建期钩子在哪注册"时给了两个位置，其一是 `nuxt.config` 的 `hooks` 节。这一篇把这一节讲透。

## hooks 节是什么

nuxt-config 的 `## hooks`：

> Hooks are listeners to Nuxt events that are typically used in modules, but are also available in `nuxt.config`. Internally, hooks follow a naming pattern using colons (e.g., build:done). For ease of configuration, you can also structure them as an hierarchical object in `nuxt.config` (as below).

三个信息：

1. 钩子是"对 Nuxt 事件的监听"，通常模块用、**nuxt.config 里也能写**。
2. 内部命名用冒号分段（`build:done`）。
3. 在 nuxt.config 里，可以写成**层次对象**——把冒号变嵌套。

## 两种写法：冒号名 vs 层次对象

冒号式（扁平）：

```ts
export default defineNuxtConfig({
  hooks: {
    'build:done' (builder) { /* ... */ },
  },
})
```

层次对象式（官方示例的写法）：

```ts
import fs from 'node:fs'
import path from 'node:path'

export default defineNuxtConfig({
  hooks: {
    build: {
      done (builder) {
        const extraFilePath = path.join(
          builder.nuxt.options.buildDir,
          'extra-file',
        )
        fs.writeFileSync(extraFilePath, 'Something extra')
      },
    },
  },
})
```

两种等价——`build: { done() {} }` 就是 `build:done` 的嵌套书写。官方称层次对象是 "For ease of configuration"，纯粹是读起来更分层的语法糖，本质还是同一个冒号名的钩子。

## 订的是哪批钩子

`nuxt.config` 的 `hooks` 节订的是 **Nuxt Hooks (Build Time)** 那批（294/297 的表：`ready`、`close`、`pages:extend`、`build:done`、`nitro:config` 等）。所以这个例子里 `build:done` 的回调拿到 `builder`（构建器实例），从 `builder.nuxt.options.buildDir` 拿生成目录、写了个额外文件——就是构建完成后干点事。

它和模块里 `nuxt.hook(name, cb)` 订的是同一批名字，只是入口不同：nuxt.config 顶层 `hooks` 节 vs 模块 setup 里 `nuxt.hook(...)`（297）。运行时钩子（`app:*`/`page:*`/`vue:*`）不在这订，那要用插件的 `nuxtApp.hook` 或组件的 `useRuntimeHook`（275/283）——别在 nuxt.config 的 hooks 节写运行时钩子，那是两拨。

## 自查

- nuxt.config 的 hooks 节写构建期钩子；`build: { done(){} }` 层次对象 = `build:done`，只是书写更分层。
- 它订的是 Nuxt Hooks (Build Time) 那批，和模块里 `nuxt.hook` 同一批名字、不同入口。
- 别把运行时钩子（app:created/page:start/vue:error）写进来——那归插件 nuxtApp.hook / 组件 useRuntimeHook。
- 冒号是钩子内部命名约定；在 nuxt.config 可扁平（`'build:done'`）也可嵌套。

## 下一篇

下一篇 361 讲 app.baseURL 与 buildAssetsDir：应用挂载路径前缀、静态资源目录前缀，以及它们的 NUXT_ 运行时覆盖。
