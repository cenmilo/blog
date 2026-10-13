---
url: /posts/f15dad0f2fb6478cd13b11a1e4503ff8/
title: 591 - debug 与 logLevel：一个没有键清单的对象与一份藏在 CLI 表里的取值
date: 2027-04-27T09:00:00+08:00
updated: 2027-04-27T09:00:00+08:00
author: cmdragon

summary:
  摘要：两节都有类型行和默认值行，但**各缺最关键的那一块**。`## debug` 说明句写着 **"Set to `true` to enable debug mode."**、当前行为是**在服务端打印钩子名与耗时、在浏览器里连钩子参数一起打**，然后补了一句 **"You can also set this to an object to enable specific debug options."**——**而这个对象里有哪些键，本节没列，整套 4.x 里也没有一处 `debug:` 的示例**；类型行还只写着 `boolean`。`## logLevel` 的默认值行是 `"info"`，正文却另给了一份**条件默认**（在 CI 或没有 TTY 时默认 `'silent'`，并且"这一项在 Vite 里作 `'silent'`、在 webpack 里作 `'none'`"），**而合法取值清单不在本节**——它在每一条 CLI 命令的选项表里：`--logLevel=<silent|info|verbose>`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 调试模式
  - 日志级别
  - 构建日志
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

排查钩子没触发、或者想看清构建每一步的顺序，会去找这两节。它们看起来记载齐全——有类型、有默认值——但正好在最该给的地方停住了。

## debug：布尔齐全，对象那一半没有清单

`## debug` 的三句：

> Set to `true` to enable debug mode.
>
> At the moment, it prints out hook names and timings on the server, and logs hook arguments as well in the browser.
> You can also set this to an object to enable specific debug options.

- **Type**: `boolean`
- **Default:** `false`

三条能确证的和三条缺口：

| 能确证 | 缺口 |
| --- | --- |
| 设 `true` 打开 debug 模式 | **对象形状有哪些键，本节没列** |
| 服务端打印**钩子名与耗时** | 打印到哪里（终端还是日志文件）没写 |
| 浏览器里连**钩子参数**一起打 | 参数打印的粒度、是否含敏感值，没写 |

注意 "At the moment" 这个措辞：官方自己把这句限定成**当前行为**，也就是说这段描述不是长期契约，版本间可能变。

再看类型行只写 `boolean`——**这与第三句 "You can also set this to an object" 直接冲突**：既然能传对象，类型就不是纯布尔。同一类缺口在 `typescript.typeCheck`（类型写布尔、正文给 `build`）和 `devServer.https`（类型写布尔、示例给对象）上都出现过，**这是配置参考里一种重复出现的形状不完整，读这几节时不能只信类型行。**

最后一句 "to enable specific debug options" 意味着对象形式是为了**只开其中一类**调试输出，但**具体哪几类、键名是什么，本节完全没给**。整套 4.x 里也**找不到一处写 `debug:` 的示例**——所以这条写法在本系列里没有可照抄的依据，要做得去类型定义里看有哪些键。

## logLevel：默认值有两份，取值清单在别处

`## logLevel` 全文：

> Log level when building logs.
>
> Defaults to 'silent' when running in CI or when a TTY is not available.
> This option is then used as 'silent' in Vite and 'none' in webpack

- **Type**: `string`
- **Default:** `"info"`

这一节的三份信息彼此是有条件的：

1. **默认值是 `"info"`**（默认值行）。
2. **但在 CI 里、或者没有 TTY 时，默认变成 `'silent'`**。所以"我没设为什么什么都不打"在流水线里是正常表现，不是配置丢了。
3. **这一项会被翻译给构建器**：在 Vite 侧作 `'silent'`，在 webpack 侧作 `'none'`。这两份是**构建器各自的日志级别词汇**，不等于 Nuxt 这一项接受 `'none'` 这个取值。

那这一项到底接受哪些字符串？**本节没有清单**——只出现过 `'info'` 与 `'silent'` 两个值。清单藏在每一条 CLI 命令的选项表里，那一行的写法是：

```
--logLevel=<silent|info|verbose>
```

`nuxt build`、`nuxt dev`、`nuxt analyze`、`nuxt add` 这些命令的选项表里都有这一条。**也就是说 CLI 侧给了三个合法取值：`silent`、`info`、`verbose`。** 配置项与这个 CLI 选项同名，取值集合按这一份读最合理，但**文档没有一句明确说 "`logLevel` 配置的可选值就是这三个"**，所以严格讲这仍是跨页推断。

## 两节配合使用的读法

一个常见的排错组合是这样的：`logLevel` 控制**构建日志的强度**（这一项在 CI 里会被自动压到 `'silent'`），`debug` 控制**钩子级别的过程输出**（服务端打钩子名与耗时、浏览器打参数）。

所以如果构建日志一片安静，先查 `logLevel` 的默认条件；如果想知道某个钩子到底有没有跑、跑在第几步，那是 `debug` 那一侧的事——**它输出的正是钩子名与耗时**，与配置里能挂钩子的那一节直接对应。

## 自查

- `## debug`：`boolean` ＋ `false` 齐全，但**正文给了"也可以传对象"这第三种取值而对象键清单缺失**，类型行也没反映；**全库无 `debug:` 示例**。行为描述被官方自己限定为 "At the moment"。
- `## logLevel`：`string` ＋ 默认 `"info"`，**另有条件默认**（CI 或无 TTY 时为 `'silent'`），并声明会被映射（Vite `'silent'`／webpack `'none'`）。
- **合法取值清单不在配置参考本节**，在 CLI 选项表：`silent|info|verbose`。
- `'none'` 是 webpack 侧词汇，**不能据此认为本项接受 `'none'`**。
- 两节分工：`logLevel` 管构建日志强度，`debug` 管钩子名／耗时／参数的输出。

## 下一篇

下一篇 592 看 `## test`：**两句记载（"Whether your app is being unit tested." ＋ `boolean` 默认 `false`），但整套 4.x 里没有一处把它读出来用**——`nuxt.options.test` 这个写法全库零命中，唯一与"测试模式"有关的第二处记载反倒是特性页里 `devLogs` 那句 "when test mode is not active"。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## debug` 三句原文、`boolean` 与 `false`、"You can also set this to an object to enable specific debug options." 那句；`## logLevel` 说明句、条件默认与 Vite/webpack 映射两句、`string` 与 `"info"`；`### typescript.typeCheck` 与 `### devServer.https` 两处同类形状缺口用于对照）
- https://nuxt.com/docs/4.x/api/commands/build 等各命令页的选项表（`--logLevel=<silent|info|verbose>` 一行的写法与描述）
