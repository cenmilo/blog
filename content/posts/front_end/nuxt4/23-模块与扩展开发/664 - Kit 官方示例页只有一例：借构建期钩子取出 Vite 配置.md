---
url: /posts/426178b3cb127fc85de386306d5a1221/
title: Kit 官方示例页只有一例：借构建期钩子取出 Vite 配置
date: 2027-05-21T10:00:00+08:00
updated: 2027-05-21T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：Kit 有一页不列函数、只放例子，而**这一页目前只有一个示例**："如何拿到 Nuxt 正在用的 Vite（或 webpack）配置"。它值得逐行读，因为这段代码里塞了四个技巧：**`loadNuxt` 传 `{ dev: false, overrides: { ssr: false } }`**、**在 `vite:extend` 钩子里先 `resolve(config)` 再 `throw new Error('_stop_')` 打断构建**、**在 `buildNuxt(...).catch` 里用错误文本是否含 `_stop_` 来区分"自己停的"与"真失败了"**、**`finally` 里调 `nuxt.close()`**。页面还点了三个已经在这么做的上游项目，并说**同样的思路可以拿去取 webpack 配置**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 编程式用法
  - Vite配置
  - 钩子
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

工具函数那一套看完，官方留了一页"别人是怎么用的"。这一页只有一个例子，但它是全套文档里少见的完整程序——**从加载 Nuxt 到把它关掉，一段写完**。

## 这一页的范围

页标题：**Nuxt Kit 工具在用法上的示例**。

**这一节标题是"Accessing Nuxt Vite Config"，也是整页唯一一节**（ Kit 其余每一页都在列函数签名，这页不讲签名，只给一段能跑的代码）。

它的适用对象写得很具体：**如果你在做一个集成，需要访问 Nuxt 所使用的运行期 Vite 或 webpack 配置，那么可以用 Kit 的工具把它取出来**。

页面还给了三个已经在这么做的真实项目作为参照：**histoire**、**nuxt-vitest**、**@storybook-vue/nuxt**（三个都直接链到各自的源码文件）。最后一句补充：**下面是一个如何从项目里访问 Vite 配置的简短示例；你可以用类似的做法去取 webpack 配置**。

## 整段代码

```js
import { buildNuxt, loadNuxt } from '@nuxt/kit'

// https://github.com/nuxt/nuxt/issues/14534
async function getViteConfig () {
  const nuxt = await loadNuxt({ cwd: process.cwd(), dev: false, overrides: { ssr: false } })
  return new Promise((resolve, reject) => {
    nuxt.hook('vite:extend', (config) => {
      resolve(config)
      throw new Error('_stop_')
    })
    buildNuxt(nuxt).catch((err) => {
      if (!err.toString().includes('_stop_')) {
        reject(err)
      }
    })
  }).finally(() => nuxt.close())
}

const viteConfig = await getViteConfig()
console.log(viteConfig)
```

**四个动作按顺序读**，下面逐个拆开。

## 第一步：加载一个"不服务端渲染"的 Nuxt

```js
const nuxt = await loadNuxt({ cwd: process.cwd(), dev: false, overrides: { ssr: false } })
```

三个入参各对应一件事：`cwd` 取当前工作目录、**`dev: false` 表示不按开发模式加载**、**`overrides` 把 `ssr` 关掉**。

这里能看到 `loadNuxt` 那条"底层是 c12，所以接受 `c12.loadConfig` 的同款选项"的用法：**`overrides` 正是那一族里的键，作用是压在所有层之上**（连项目自己的 `nuxt.config` 也被它盖住）。

**为什么要关 SSR、为什么 `dev` 取 `false`，这一页没有解释**——它属于这段代码作者的取值选择，读者照抄前应该知道：**`dev: false` 会让它走生产形态的配置合并**，与开发服务器下拿到的配置可能不同。

## 第二步：在钩子里既交付配置、又叫停构建

```js
nuxt.hook('vite:extend', (config) => {
  resolve(config)
  throw new Error('_stop_')
})
```

这段是整页最巧妙的一处，**两行做了相反方向的事**：**`resolve(config)` 把想要的东西交付出去**，紧接 **`throw new Error('_stop_')` 打断接下来的构建**。

为什么要打断：目的是取配置，**不是真的要打包**。`vite:extend` 是"允许扩展 Vite 默认上下文"那个构建期钩子，在这个时点上配置已经成型，所以此时既拿到了值、也还来得及止损。

**钩子只能靠抛错来中断流程**这件事在这里被当工具用了——而抛出的那个 `Error` 是**给下一步当标记用的**，不是真错误。

## 第三步：用错误文本区分自停与真失败

```js
buildNuxt(nuxt).catch((err) => {
  if (!err.toString().includes('_stop_')) {
    reject(err)
  }
})
```

`buildNuxt` 的 promise 一定会因为那个 throw 而拒绝，所以必须判：**错误文本里含 `_stop_` 就当没事发生，不含才 `reject`**。

**这就是"用一个魔法字符串当控制信号"的代价：判据是文本匹配**。页面写的 `err.toString().includes('_stop_')` 就是这个形状。**如果哪天钩子里抛出的错误信息不含这段文本（或真实错误恰好含这段文本），这套逻辑就会判错**——照抄时值得把这个标记换成更不易撞的值。

顺带一提，代码第一行上面那条注释指向 **nuxt/nuxt 的 issue 14534**，也就是这套绕法的由来讨论，**页面本身没有解释这个 issue 的内容**。

## 第四步：无论如何都要关掉实例

```js
}).finally(() => nuxt.close())
```

`finally` 里调 **`nuxt.close()`**——成功取到配置也好、失败 reject 也好，**这个 Nuxt 实例都会被关闭**。

这一行与构建期钩子里那条 `close` 是同一件事的两端：`close` 钩子给模块用（**清理你打开的、持有的、启动的 watcher**），而这里的 `nuxt.close()` 是**编程式用法里主动触发那次关闭的调用**。

## 这例子的可移植性与边界

| 想要什么 | 改动 |
| --- | --- |
| Vite 配置 | 原样，钩子用 `vite:extend` |
| webpack 配置 | **页面说用类似做法**，对应钩子是 `webpack:config`（"在配置 webpack 编译器之前调用"） |
| 已经解析完的配置 | 读 resolved 的那两个钩子——**但它们在 Nuxt 5+ 已被标弃用** |

第三行这个提醒有依据：这一页用的是 `vite:extend`（未被标弃用），而 `vite:extendConfig` 与 `vite:configResolved` 两个在构建器那一节里都挂着 "Deprecated in Nuxt 5+"。**同一个 Vite 一族里，有的钩子在退场、有的还在**，所以照抄时别把钩子名换成那两个。

这一页没有讲的：**`vite:extend` 回调参数对象的完整形状**（`config` 之外还有什么字段）、**取 webpack 配置的具体代码**（只说"类似做法"）、**这段代码该放在哪里运行**（脚本？测试？另一个插件里？）、以及**顶层 `await` 需要什么前提**（示例直接写 `const viteConfig = await getViteConfig()`，没提模块或 ESM 要求）。

下一篇开始讲钩子本身——三张表怎么分、挂法有几种、以及自定义钩子要动哪三个接口。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/examples
