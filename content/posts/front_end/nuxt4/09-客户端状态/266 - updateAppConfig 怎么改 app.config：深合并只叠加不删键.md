---
url: /posts/363aeac2106409de7be00b9e97a03f3f/
title: updateAppConfig 怎么改 app.config：深合并只叠加不删键
date: 2027-01-08T11:00:00+08:00
updated: 2027-01-08T11:00:00+08:00
author: cmdragon

summary:
  摘要：app.config.ts 里的值是构建期定死的，但运行时的主题、开关常常要变。改它用的是 useAppConfig() 读、updateAppConfig() 写这一对。这一篇聚焦 updateAppConfig 的一个具体行为——它做的是"深合并"（deep assignment），已有的嵌套属性会被保留，你能加键、能改键，但没法用它删掉一个键。理解这一点，才不会写出"更新后旧键还在"这种让人困惑的结果。同时把两个页面上的示例、类型名（AppConfigInput / AppConfig / SharedAppConfig）交代清楚。

categories:
  - 前端开发

tags:
  - nuxt4
  - updateAppConfig
  - useAppConfig
  - app.config
  - 响应式
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇说 `app.config.ts` 是构建期定死的。但 UI 上的东西常常要运行时变：用户切了主题色、某个开关被打开。Nuxt 给的这一对是：`useAppConfig()` 读、`updateAppConfig()` 写。这篇只讲后者那一个关键行为——它到底怎么"改"，因为改的方式有个不那么直觉的地方。

## 读和写这一对

useAppConfig 那一页很短，就是把当前 app.config 拿出来：

```ts
const appConfig = useAppConfig()
console.log(appConfig)
```

updateAppConfig 那一页给的完整示例，正好演示"写完能立刻在同对象上读到新值"：

```js
import { updateAppConfig, useAppConfig } from '#imports'

const appConfig = useAppConfig() // { foo: 'bar' }

const newAppConfig = { foo: 'baz' }
updateAppConfig(newAppConfig)

console.log(appConfig) // { foo: 'baz' }
```

注意最后一行：`updateAppConfig` 没有返回值让你去接收，它是**就地更新** `useAppConfig()` 拿到的那个响应式对象。这也是 app.config 那页 usage 示例的写法——`appConfig` 先拿到 `{ foo: 'bar' }`，`updateAppConfig({ foo: 'baz' })` 之后，同一个 `appConfig` 变量打印出来就是 `{ foo: 'baz' }`。因为是响应式的，模板里用到它的地方会跟着重渲染。

## 它的更新方式是"深合并"

这一句是全篇的核心，原文在 updateAppConfig 页的 note：

> Updates the `app.config` using deep assignment. Existing (nested) properties will be preserved.

"deep assignment"（深赋值/深合并）意味着两件事：

1. **它是逐层往里合并的，不是整体替换。** 假设你的 app.config 是：

```ts
export default defineAppConfig({
  theme: {
    primaryColor: '#ababab',
    radius: '4px',
  },
})
```

你只想改主色：

```ts
updateAppConfig({ theme: { primaryColor: '#ff0000' } })
```

深合并的结果是 `{ theme: { primaryColor: '#ff0000', radius: '4px' } }`——`radius` 被"保留"，这正是 note 那句 "Existing (nested) properties will be preserved" 在说的事。如果它做的是浅替换，`theme` 整个被你传的对象换掉，`radius` 就没了。深合并帮你避免了这种"改一个、丢一片"。

2. **它只能加和改，删不掉键。** 合并是把新值盖到旧值上，旧值里有、你没提到的键会原样留着。想"移除某个配置键"，靠 `updateAppConfig` 做不到——文档没有给它任何删除语义。真要让某个值变成"无效"，只能把它显式改成一个代表空的值（比如 `''`、`null` 或某个默认），而不是指望它消失。

这两点合起来记：**传进去的是"要覆盖/新增的部分"，其余全部原样保留；它不会动你没写的键，也不会因为你没写就删掉旧键。**

## 什么时候用

app.config 那一页把可更新的时机写清楚了——"at runtime within lifecycle or using a nuxt plugin and editing it with HMR (hot-module-replacement)"。翻译成人话，`updateAppConfig` 的典型落点是：

- 在**生命周期**里响应某个事件改配置；
- 在**插件**里初始化或改配置；
- 开发时改 app.config 文件，靠 HMR 生效。

它不是给你存"会频繁变的业务数据"用的——那是 `useState`（251）的活。app.config 是配置容器，只是这份配置恰好是响应式的、可以在运行时被 `updateAppConfig` 调整。

## 三个类型名别混

app.config 那页的"Typing"部分出现三个接口名，容易看花，理一下：

| 类型名 | 用在哪 | 作用 |
| --- | --- | --- |
| `AppConfigInput` | 模块作者声明"允许往里写什么输入" | 约束 `updateAppConfig` / 写入时的输入形状，**不影响 `useAppConfig()` 的返回类型** |
| `AppConfig` | 应用代码里 `useAppConfig()` 的返回类型 | 手写它要当心——官方 warning："Be careful when typing `AppConfig` as you will overwrite the types Nuxt infers from your actually defined app config." |
| `SharedAppConfig` | 应用代码之外（服务端路由、`shared/`、`nuxt.config`） | 那些上下文里 `useAppConfig()` 返回的是 `SharedAppConfig`，要跨边界有类型就扩展它 |

对新手来说，默认情况下不用手写类型——Nuxt 会从你写的 app.config 自动推。只有在跨应用边界（服务端路由里读不到 app 侧推的类型、那里是 `unknown`）才需要去扩 `SharedAppConfig`。

## 自查

- 用 `updateAppConfig` 改嵌套配置时，记住它是深合并：你只传要改的那一小块，同级没写的键会保留。
- 别指望它能删键——它只叠加，删不掉已有属性。
- 它是就地更新响应式对象，不返回新对象，所以写完直接读 `useAppConfig()` 就能看到。
- 要放的是"配置"（主题、开关），不是"会变的业务数据"（那个用 `useState`）。

## 下一篇

下一篇 267 讲 `stripNeverHydratedData` 这个实验开关：它把"永不水合的组件"的数据从 payload 里剔出去，是 264 里"少往 payload 塞东西"的一个自动化手段。

## 本文依据

- updateAppConfig（`api/utils/update-app-config`）：note "Updates the `app.config` using deep assignment. Existing (nested) properties will be preserved."；`{ foo: 'bar' } → updateAppConfig({ foo: 'baz' }) → { foo: 'baz' }` 示例。
- useAppConfig（`api/composables/use-app-config`）：定位 "Access the reactive app config defined in the project." 与 `const appConfig = useAppConfig()` 用法。
- app.config.ts（`directory-structure/app/app-config`）：更新时机 "at runtime within lifecycle or using a nuxt plugin and editing it with HMR"；usage 里 `updateAppConfig(newAppConfig)` 就地改变 `appConfig` 的示例；Typing 节 `AppConfigInput`（"This will not affect the type of `useAppConfig()`"）、`AppConfig`（overwrite 警告）、`SharedAppConfig`（应用外返回类型）。
