---
url: /posts/30cc89447cfab15ae1f96eadbe026936/
title: App 运行时钩子头五个：两个 error 钩子共用一句描述
date: 2027-05-22T09:00:00+08:00
updated: 2027-05-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：运行时表开头五行是**应用外壳与错误的最早几个点**：`app:created`（初始 `vueApp` 实例被创建时，参数是 `vueApp`）、`app:error`（参数 `err`）、`app:error:cleared`（参数 `{ redirect? }`）、`vue:setup`（无参数，**且原话要求"This callback must be synchronous"**）、`vue:error`（参数 `err, target, info`，触发条件是"Vue 错误冒泡到根组件"）。**这一组里有一处表格错误：`app:error` 与 `app:error:cleared` 的描述逐字相同**，都写着"发生致命错误时调用"——**区别只能从参数形状上看**。还有一点实用价值：**这五行的 Environment 全部是 `Server & Client`**，也就是同一份插件代码里它们在两边都会跑。

categories:
  - 前端开发

tags:
  - nuxt4
  - 运行时钩子
  - 错误处理
  - 插件
  - vueApp
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

运行时那张表有 20 行，头五行是"应用刚建起来、以及出错的时候"。这一组里有一条唯一的硬性约束，也有一处表格自身的错误。

## 五行照录

| 钩子 | 参数 | 环境 | 描述（原文译法） |
| --- | --- | --- | --- |
| `app:created` | `vueApp` | Server & Client | 初始的 `vueApp` 实例被创建时调用 |
| `app:error` | `err` | Server & Client | 发生致命错误时调用 |
| `app:error:cleared` | `{ redirect? }` | Server & Client | 发生致命错误时调用 |
| `vue:setup` | - | Server & Client | **Nuxt 根节点的 setup 初始化时调用；这个回调必须是同步的** |
| `vue:error` | `err, target, info` | Server & Client | 当一个 Vue 错误冒泡到根组件时调用 |

**"描述"那一列最后两行的原文里各多了一句**：`vue:setup` 后面跟着 "This callback must be synchronous."，`vue:error` 后面挂着一个 Learn More 链接指向 Vue 文档的 onErrorCaptured 一节。

## 先记住那条同步要求

**五行里只有 `vue:setup` 带强制约束**：**这个回调必须是同步的**。

它的位置也解释了为什么要同步——**它对应的是"Nuxt 根节点的 setup 被初始化"这一刻**。这个时机上应用还在装配中，异步的东西赶不上后续步骤。

**违反之后会怎样，表格里没有写**（不报错？静默失效？），所以这条只能当作"必须遵守但不要指望它提醒你"的规则：**要在这个时机做检查，先看自己有没有 `await` 或返回 Promise**。

## 两个 error 钩子的描述是同一句

`app:error` 与 `app:error:cleared` 在表里的描述**逐字相同**，都是"发生致命错误时调用"。

**这处是文档表格自身的错误**：第二个钩子的名字读起来是"错误被清除"，与描述对不上；而页面**没有为这一处提供任何其他说明**，也没有 Environment 之外的信息可依据。

所以此刻能确定地用的只有参数形状：

| 钩子 | 参数 | 从参数能读出什么 |
| --- | --- | --- |
| `app:error` | `err` | 给你一个错误对象 |
| `app:error:cleared` | `{ redirect? }` | 给你一个**带可选 `redirect` 键的对象** |

`redirect` 后面的问号在表里表示**这个键可选**（同一列里 `keys?`、`pageComponent?` 用的是同样写法）。**它具体控制什么、清除错误时为何需要一个重定向目标，文档未说明**。要在这个钩子上做"错误页关闭后跳去哪"的动作，只能自己实测。

## 参数列怎么读

这一列不是类型，而是形状速记，四种写法各有含义：

| 写法 | 出现在 | 含义 |
| --- | --- | --- |
| `-` | `vue:setup` | 没有参数 |
| `vueApp` | `app:created`、`app:beforeMount`、`app:mounted` | 一个参数 |
| `err` | `app:error` | 一个参数 |
| `{ redirect? }` | `app:error:cleared` | 一个对象，含可选键 |
| `err, target, info` | `vue:error` | **逗号分隔的多个参数**（不是数组、不是解构） |

**`vue:error` 是唯一一个用逗号列出多个参数的**，这与 `err, path` 那类构建期钩子写法一致——**这一列里逗号表示"依次传多个实参"**。

## 环境列在这一组还不重要

五行 Environment 全是 `Server & Client`，所以**这组看不出环境差别**。它真正开始起作用是在下一组（`app:rendered` 只在 Server、`app:mounted` 只在 Client）。

不过"两侧都会跑"本身有实操含义：**同一份插件代码里，这些钩子在服务端和浏览器各触发一次**。所以**在这些回调里累加计数器、push 数组这类动作，会把同一份逻辑执行两遍**——这属于写副作用时的常识，而不是文档给出的警告（**表格里没有任何一句提到这点**）。

## `vue:` 与 `app:` 两个前缀并存

这一组里既有 `app:` 开头也有 `vue:` 开头，读法可以从 `vue:error` 那句描述出来：**它讲的是"Vue 的错误冒泡到根组件"**，链接也指向 Vue 的 `onErrorCaptured`——**`vue:` 一族对应 Vue 层的事件，`app:` 一族对应 Nuxt 应用外壳的事件**。

`app:created` 的参数正好是 `vueApp`，把两层的先后也串起来了：**先有 `vueApp` 实例被创建（`app:created`），随后根节点 setup 时 `vue:setup` 被调用**。

## 用在哪、怎么验

| 想做的事 | 这一组里对应的钩子 |
| --- | --- |
| 拿到 `vueApp` 就往下挂东西 | `app:created` |
| 处理致命错误（应用级，不是路由级） | `app:error` |
| 在根 setup 阶段做同步注入 | `vue:setup`，**注意不能有异步** |
| 捕获冒泡到根的 Vue 错误 | `vue:error`（对照 Vue 的 onErrorCaptured） |

挂法用运行时那一种：**在插件里 `nuxtApp.hook('...')`**（服务端与客户端两侧都跑，所以插件是 `mode: 'all'` 或者干脆不限定 mode）。

这一组未记载的还有：`app:created` 与 `app:beforeMount` 之间的完整顺序、`app:error` 触发后错误页怎么出现（**错误页那一章讲的是另一套东西**，本页没有把它和这两个钩子连起来）、以及 `vue:setup` 的"同步"具体校验到哪一层。

下一篇读六到十二行，环境列在那一组开始决定你在哪一侧看到日志。

## 本文依据

- https://nuxt.com/docs/4.x/api/advanced/hooks
