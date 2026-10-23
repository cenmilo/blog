---
url: /posts/403c20342d0a2f00c33f08ae6bf581da/
title: 467 - parseErrorData：把结构化错误读回来
date: 2027-03-16T11:00:00+08:00
updated: 2027-03-16T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`createError({ data: {...} })` 放进去的结构化信息，是错误处理里唯一能穿过响应的容器（462 记过：API 路由里的 `message` 不会传播到客户端，能到的是短 `statusText` 和 `data`）。`experimental.parseErrorData` 管的是这条链上最后一段——官方只给它一句定性："Whether to parse `error.data` when rendering a server error page"，**默认开启**，可设 `false` 关掉。限定词是 **when rendering a server error page**：它的作用对象是服务端渲染错误页那一次。除了这一句，官方没有示例、没有解释关掉之后 `error.data` 是什么形态、也没说什么场景该关（文档未说明）。

categories:
  - 前端开发

tags:
  - nuxt4
  - parseErrorData
  - 错误数据
  - createError
  - 错误页
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

接口返回的错误里带了一个字段清单：哪个表单字段不合法、规则版本号是什么。你在 `error.vue` 里读 `error.data`，读到的是字符串而不是对象。

这一篇讲的就是这一件事：**`error.data` 会不会被解析回来**，以及官方对它的记载到底有多少。

## 官方记载只有这些

`parseErrorData` 整节是这样：

> Whether to parse `error.data` when rendering a server error page.
>
> This flag is enabled by default, but you can disable this feature:

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    parseErrorData: false,
  },
})
```

三句话，能确定的只有两点：

- **默认开启**（官方用的是这一节里那句标准的"This flag is enabled by default"）。
- **作用时机被限定在"服务端渲染错误页"这一件事上**。

而关掉之后 `error.data` 呈现成什么（原始字符串？`undefined`？）、什么场景该关、它和客户端读到的那份 `data` 是不是同一套解析——**这三项文档一个字都没写**。所以这一篇剩下的部分只讲能查证的东西：`data` 这条链的其他几环。

## data 的四个落点

前面几篇已经分别记过，这里排成一条链：

| 环节 | 官方原话要点 | 出处 |
| --- | --- | --- |
| 写入 | `createError` 的参数对象里可以带 `data` | 462（`api/utils/create-error`） |
| 自定义字段必须走这里 | "If you have an error with custom fields they will be lost; you should assign them to `data` instead" | 464（`directory-structure/app/error`） |
| 客户端从接口错误里读 | "Alternatively, you can use the `data` property to pass data back to the client. When handling the error with `useFetch`, the custom data is available at **`error.value.data.data`**" | 462（`api/utils/create-error`） |
| 错误页里读 | `NuxtError` 接口里的 `data?: unknown`；服务端渲染错误页时是否解析由 `parseErrorData` 决定 | 464 + 本篇 |

`error.value.data.data` 那两层 `.data` 是这条链上最容易漏的一层：**外层是错误对象的 `data`（h3 那侧的包装），内层才是你放进 `createError({ data })` 的那个值**。写测试或做表单错误回显时按这个嵌套取。

对照一条 462 记的规则：接口里 `message` 不传播到客户端、短 `statusText` 可达——**所以要给用户的结构化信息只有一个合法容器，就是 `data`**。`parseErrorData` 管的正是这个容器在错误页那一次能不能被还原成对象。

## 一条相邻的形态差异

`data` 能不能安全地放东西，还要看 payload 那侧的序列化规则。官方在预渲染那一节写的是"Payloads are serialized with devalue, so custom types (such as class instances) need payload plugins with custom reducers and revivers to survive the round trip"（448）——那条讲的是取数结果与应用状态；**错误数据的序列化走不走同一套，文档没有写**（文档未说明）。

稳妥的判断方式是把 `data` 限制在可 JSON 化的形状里：字符串、数字、布尔、数组、普通对象。类别实例、`Map`/`Set`、`Date` 这类放进来之前，先用下面这组观察确认它们能过。

## 三分钟验证这一项有没有生效

不用猜，按顺序看三处：

1. **造一个带 `data` 的错误。** 服务端路由里按官方形状抛：

```ts [server/api/error.ts]
export default eventHandler(() => {
  throw createError({
    status: 404,
    statusText: 'Page Not Found',
    data: { fields: ['email'], ruleVersion: 3 },
  })
})
```

2. **看两条不同的读取路径。** 一条是客户端 `useFetch` 的 `error.value.data.data`（应该拿到那个对象），另一条是浏览器直接访问该页面、由服务端渲染错误页时读 `error.data`。
3. **改配置各跑一次。** 把 `parseErrorData` 设 `true` 与 `false` 各构建一次，对比第二条路径上 `error.data` 的形状（对象还是字符串、还是空）。

这一步做完，你就有了这一项在你项目里的实际行为——**官方没给的三种答案，只能用这种方式得到**。要顺带看响应体，可以加一个 `Accept: application/json` 的请求（460 记过这条分岔）。

## 自查

- `parseErrorData` 默认开，作用面被官方限定为"渲染服务端错误页时"；关掉后的形态、适用场景、与客户端读取是否同源，文档均未记载。
- 结构化错误信息只有 `data` 这一个合法容器：`message` 在接口里不传播到客户端，自定义字段不放进 `data` 就会丢。
- 客户端读接口错误要看两层：`error.value.data.data`。
- 别把复杂类型放进 `data`，官方只在 payload 侧写了 devalue 序列化，错误侧没说。
- 这一项的真实行为可以用"造错 → 两条路径读 → 开关各构建一次"三步测出来。

## 下一篇

下一篇 468 进入错误码页：官方对 `NUXT_E1001`（组合式函数在 Nuxt 上下文外调用）、`NUXT_E1006`（`onPrehydrate()` 没被构建流水线转换）与 `NUXT_E1007`（编译期宏在运行时执行）三节各写了两段——一段说清是什么触发的，一段给出修法，三处都落在"编译期处理"与"上下文"这两个概念上。
