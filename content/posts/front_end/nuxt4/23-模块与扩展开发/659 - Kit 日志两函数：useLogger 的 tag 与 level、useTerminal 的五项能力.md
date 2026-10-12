---
url: /posts/efbfef1156457e630e26511007e7a766/
title: Kit 日志两函数：useLogger 的 tag 与 level、useTerminal 的五项能力
date: 2027-05-19T11:00:00+08:00
updated: 2027-05-19T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：模块要在终端里说话有两个入口。**`useLogger(tag?, options?)` 返回一个底层用 consola 的日志实例**，`tag` 会作为后缀加在每条消息上、**显示在时间戳右侧**，选项里点名了 `level`、`reporters`、`defaults`、`formatOptions` 四项（官方示例用 `{ level: options.quiet ? 0 : 3 }` 这样的数字，但**没有给出数值表**）。**`useTerminal()` 返回的是一组与终端交互的原语**，设计目标是那句：**当 Nuxt 运行在交互宿主里（例如 `nuxt dev` 的终端 UI）时，提示、任务与通知会交给它，因此可以被应答、并集中在一处渲染；否则它们退化成日志**。它一共五项能力：**`interactive`、`withTerminal`、`prompt`、`startTask`、`notify`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 日志
  - 终端交互
  - consola
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块在构建时往终端打字是常态，但 `console.log` 有两个问题：没有别人的前缀、也没法被静音。这一页给的两个函数各解决一半。

## `useLogger`：带 tag 的日志实例

开场句：**Nuxt 提供一个日志实例供你使用，它以额外特性记录消息；`useLogger` 让你取到这个日志实例**。

```ts
function useLogger (tag?: string, options?: NuxtLoggerOptions): NuxtLogger
```

定性一句：**返回一个日志实例，底层使用 consola**。

两个参数各一句说明：

**`tag`**：**一个作为所有日志消息后缀的标签，显示在时间戳右侧**——这就是为什么用户看到的一行里会有你的模块名，而不是满屏不知出处的输出。

**`options`**：**日志器选项，例如 `level`、`reporters`、`defaults` 与 `formatOptions`**。注意这句用的是"such as"，**这一页没有给出 `NuxtLoggerOptions` 的字段表**，这四项之外还有什么、各自类型如何，要去看 consola。

基础用法：

```ts
import { defineNuxtModule, useLogger } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options, nuxt) {
    const logger = useLogger('my-module')

    logger.info('Hello from my module!')
  },
})
```

带 level 的示例给的是**用模块自己的选项决定数字**：

```ts
const logger = useLogger('my-module', { level: options.quiet ? 0 : 3 })

logger.info('Hello from my module!')
```

**`0` 与 `3` 各自对应什么，这一页一个字没提**。从写法能读出的是"quiet 时取 0、否则取 3"，也就是**数字越大越啰嗦**，但完整的级别表（哪些数字是 silent/fatal/error/warn/info/log/success/trace/debug）必须去 consola 的文档核对，**不要凭 0 和 3 这两个样本推断中间档**。

## `useTerminal`：同一段代码在两种宿主里

定性比 logger 长，因为它解释了整套行为：

> Returns a set of primitives for interacting with the user's terminal. When Nuxt is running inside an interactive host such as the `nuxt dev` terminal UI, prompts, tasks and notifications are handed to it, so they are answerable and rendered in one place. Otherwise they fall back to logging.

三处要点：**返回的是"与终端交互的原语集合"**；**在交互宿主里（举的例子就是 `nuxt dev` 的终端 UI），提示、任务、通知三样都交给宿主**，因此**可以被应答、并集中在一处渲染**；**否则它们退化成日志**。

最后那半句是这件事的实际价值：**你的模块问用户一个问题，在 `nuxt dev` 里是一个能答的提示，在 CI 里自动变成一行日志，不会卡住构建**。

类型把能力列得很清楚：

```ts
function useTerminal (): NuxtTerminal

interface NuxtTerminal {
  readonly interactive: boolean
  withTerminal: <T>(work: () => Promise<T>) => Promise<T>
  prompt: (message: string, options?: NuxtPromptOptions) => Promise<any>
  startTask: (label: string) => NuxtTerminalTask
  notify: (notification: NuxtTerminalNotification) => NuxtTerminalNotice
}
```

五项逐个看：

| 成员 | 说明 |
| --- | --- |
| `interactive` | **是否存在交互宿主。为 `false` 时，下面这些原语改为向当前进程流写日志** |
| `withTerminal` | **在 `work` 期间借用终端，暂停任何宿主 UI 并释放 `stdin`**；需要直接写终端或直接读 `stdin` 时用它；**并发调用会被串行化，而在一次借用内部的嵌套调用会立即执行** |
| `prompt` | **向用户提一个问题，并在提示打开期间借用终端**；**接受与 `logger.prompt` 相同的选项** |
| `startTask` | **启动一个长时任务，在可用时渲染到宿主的状态面板上**；用 **`task.stop(message?, outcome?)`** 结束它，或用 **`task.update(label)`** 改它的标签 |
| `notify` | **显示一条消息并把它留在屏幕上，直到用户确认或调用 `notice.dismiss()`**；**`notice.dismissed` 会在提示消失后 settle** |

`interactive` 那一行是整套设计的开关：**它是唯一一个明确说明"降级路径"的字段**——其余四项在 `false` 时都按它说的走日志。

用法示例是"装不装这个包"这一类一次性询问加一段耗时操作：

```ts
import { defineNuxtModule, useTerminal } from '@nuxt/kit'

export default defineNuxtModule({
  async setup () {
    const terminal = useTerminal()

    const install = await terminal.prompt('Do you want to install `my-module`?', { type: 'confirm' })
    if (!install) {
      return
    }

    const task = terminal.startTask('Installing `my-module`...')
    // run your actual task
    await runPackageInstall('my-module')
    task.stop('Installed `my-module`')
  },
})
```

三处写法值得照抄：**`prompt` 传 `{ type: 'confirm' }` 并把返回值直接当布尔判断**（返回类型是 `Promise<any>`，示例把它当真假用）、**`startTask` 拿到的是任务句柄、真正干活之后才 `task.stop(...)`**、**`stop` 带一句完成文案**。

## 两者各用在什么时候

| 你要做的 | 用哪个 |
| --- | --- |
| 输出普通信息、警告、错误 | `useLogger(模块名)`，让消息带上你的 tag 且可被 level 静音 |
| 问用户一个需要回答的问题 | `useTerminal().prompt` |
| 有一段耗时动作要显示进度 | `useTerminal().startTask` ＋ `task.stop` |
| 要显示一条等确认的消息 | `useTerminal().notify` |
| 直接读写终端或 `stdin` | `useTerminal().withTerminal` |

**logger 是"我说你听"，terminal 是"可能要你回应"**——这条分界决定了你的模块在 CI 里会不会卡住。

这一页没写的还有：`level` 的数值含义与默认值、`NuxtLoggerOptions` 除点名的四项之外有什么、`notify` 的入参 `NuxtTerminalNotification` 有哪些字段、`prompt` 的返回类型（`Promise<any>`）在各种 `type` 下分别是什么形状、以及这两个函数是否受"Kit 只给模块用"那条限制（**按 Kit 总页的规则它们是构建期工具，运行时代码里不该出现**）。

另有一处衔接值得记：`useTerminal` 与 DevTools 或 `nuxt dev` 的终端 UI 有关，**而终端 UI 的行为由 CLI 侧提供**，所以同一段代码在 `nuxt build` 下走的就是那条"退化成日志"的分支。

下一篇讲构建器那一页的四个函数，其中 `extendViteConfig` 已被标为弃用。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/logging
