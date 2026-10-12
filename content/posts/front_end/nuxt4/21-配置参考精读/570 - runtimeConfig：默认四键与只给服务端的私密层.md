---
url: /posts/1e7fa37ac64a0647d0de90a09d0c1d63/
title: runtimeConfig：默认四键与只给服务端的私密层
date: 2027-04-20T09:00:00+08:00
updated: 2027-04-20T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`runtimeConfig` 这一节的正文只有五句，加一行 `**Type**: object` 和一份默认值 JSON。五句话里藏着这个键的核心规则：**整个对象默认只有服务端拿得到，两个键名是例外**——写在 `public` 和 `app` 下面的内容会一起发到前端，所以密钥不能放进这两个。默认值 JSON 给的是 `public: {}` 和 `app` 下面的四个键（`buildId`、`baseURL`、`buildAssetsDir`、`cdnURL`），**这四个是框架自己填的**。环境变量覆盖那句只给了两个例子，**映射算法这一节没写**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 运行时配置
  - 环境变量
  - 密钥管理
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

要调外部服务，密钥该放哪？写进前端代码会被打进包里，写成硬编码上库就泄露。`runtimeConfig` 就是为这件事存在的，但它那一节只有五行字，最容易读漏的恰好在第二句和第四句之间。

## 这一节的正文全在这里

官方 `## runtimeConfig` 的记载，逐句：

> Runtime config allows passing dynamic config and environment variables to the Nuxt app context.

> The value of this object is accessible from server only using `useRuntimeConfig`. It mainly should hold *private* configuration which is not exposed on the frontend. This could include a reference to your API secret tokens.

> Anything under `public` and `app` will be exposed to the frontend as well.

> Values are automatically replaced by matching env variables at runtime, e.g. setting an environment variable `NUXT_API_KEY=my-api-key NUXT_PUBLIC_BASE_URL=/foo/` would overwrite the two values in the example below.

然后是一行类型和一份默认值，没有别的。

第二句和第四句挨着读会有点别扭：先说"这个对象只能从服务端用 `useRuntimeConfig` 拿到"，紧接着说"`public` 和 `app` 下面的任何东西前端也拿得到"。这两句不矛盾，读法是：**这个对象默认是私密的，`public` 与 `app` 这两个键名是显式的暴露口子。** 也就是说边界不是按文件分的，而是按你写在哪个键下面分的。

## 默认值 JSON：app 下面那四个键不是你写的

官方给的 `**Default**`：

```json
{
  "public": {},
  "app": {
    "buildId": "4a2e2d30-418f-41df-8e58-ed5df06de7fd",
    "baseURL": "/",
    "buildAssetsDir": "/_nuxt/",
    "cdnURL": ""
  }
}
```

| 键 | 默认值 | 这一节里说了什么 |
| --- | --- | --- |
| `public` | `{}` | 空对象；只有那句"底下的东西会发到前端" |
| `app.buildId` | 一个 UUID 字面量 | 未作说明 |
| `app.baseURL` | `"/"` | 未作说明 |
| `app.buildAssetsDir` | `"/_nuxt/"` | 未作说明 |
| `app.cdnURL` | `""` | 未作说明 |

两件事要记住：

一是 **`app` 不是留给你放业务配置的**。它是框架自带的四个键，你写 `runtimeConfig.app` 只会覆盖这四项的行为（`baseURL`、`buildAssetsDir`、`cdnURL` 这三个怎么影响资源路径，属于另外的话题）；把你的密钥或主题值塞进 `app` 里，等于塞进一个会发到前端的键名下——第四句就是这么规定的。

二是 `public` 默认是空的。前端要读的东西必须自己显式放进 `public`，**没放就得在服务端路由里读**。

## 示例：官方只写了两个键

`**Example**:` 那段也是这一节的全部内容之一：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    apiKey: '', // Default to an empty string, automatically set at runtime using process.env.NUXT_API_KEY
    public: {
      baseURL: '', // Exposed to the frontend as well.
    },
  },
})
```

注意这个形状：**私密键直接写在顶层**（`apiKey`），**公开键写在 `public` 下面一层**（`public.baseURL`）。按上一节那条规则，`apiKey` 与 `public.apiKey` 处在两个不同位置——后者属于会发到前端的 `public` 之下，前者不属于。

默认值给空字符串而不是省略，注释里写得很清楚："Default to an empty string, automatically set at runtime using `process.env.NUXT_API_KEY`"。**这一行是把环境变量名字直接写在配置里当文档用**，让人一眼看见缺哪个变量。

## 环境变量覆盖：这一节只举了两个例子

那句 "Values are automatically replaced by matching env variables **at runtime**" 给的对应关系是：

| 环境变量 | 覆盖掉的键 | 出处 |
| --- | --- | --- |
| `NUXT_API_KEY=my-api-key` | `apiKey` | 本节的例子 |
| `NUXT_PUBLIC_BASE_URL=/foo/` | `public.baseURL` | 本节的例子 |
| `NUXT_APP_CDN_URL` | `app.cdnURL` | useRuntimeConfig 那页 |

能确定的只有这些：**前缀是 `NUXT_`**、**下划线对应点号层级**（`PUBLIC_BASE_URL` → `public.baseURL`，`APP_CDN_URL` → `app.cdnURL`）、**发生在运行时**。

而这几个问题在本节**一个字都没写**：

- 驼峰键名要怎么变成环境变量名（`apiSecretToken` 是 `NUXT_API_SECRET_TOKEN` 还是 `NUXT_API_SECRETtOKEN`）。
- **连续大写的键**怎么映射。
- 嵌套更深的 `public` 结构能不能覆盖、能不能新增只存在于环境变量里的键。
- 类型层面：读一个配置里没声明、只由环境变量提供的键，TS 会不会报错。

这几条要实测，本节没有依据。另外"at runtime"这个时机和 `.env` 文件的读取窗口不是一回事——配置文件 `.env` 只在 dev/build 阶段被读，**跑构建产物时不读**，所以生产上这些变量要在托管平台的环境变量里注入。

## 三个容易写错的地方

**把密钥放进 `public`。** 表现是前端能读到了、接口也通了，但密钥在浏览器可见——`public` 下面的键就是设计成前端要读的，`api/composables/use-runtime-config` 那页对私密键的说法是 "only accessible on the server"，反过来 `public` 里的值前后端都能拿到。按第四句的规则，`public` 与 `app` 之下的内容一律发到前端，没有例外开关。

**以为私密键在客户端读得到。** 在服务端路由里 `useRuntimeConfig()` 能读到全部；这一节写的就是"from server only"。客户端读私密键的返回值不要靠猜——真要确认，在页面里读一次打印出来看。

**指望构建期的值等于运行期的值。** 覆盖发生在运行时，配置里那份默认值才是构建期能看见的东西。CI 里构建通过、线上读到空字符串，通常是这个时机差造成的。

## 自查

- `runtimeConfig` 一节 = 五句正文 + `Type: object` + 默认值 JSON + 一个两键示例，**没有更多**。
- 边界按键名分：顶层与自定义子对象私密；**`public` 和 `app` 之下会发到前端**。
- 默认值里 `app` 的四个键（`buildId`、`baseURL`、`buildAssetsDir`、`cdnURL`）是框架自带的，`public` 默认空对象。
- 环境变量覆盖：`NUXT_` 前缀、下划线点号对应层级、发生在 **runtime**；**命名映射算法本节未写**。
- 密钥放顶层或 `public` 之外的自定义键；`.env` 生产不读，生产靠平台注入。

## 下一篇

下一篇 571 看 `appConfig`：这一节只有两句和一个**完全空的 `### nuxt` 子项**，真正的内容全在 `app/app.config.ts` 那一页——包括一句和这一节直接相关的：**写在 `nuxt.config` 里的 `appConfig` 键，在所有上下文里都有类型，而文件里定义的键在服务端只被推成 `unknown`**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## runtimeConfig` 五句正文原文、`**Type**: object`、默认值 JSON 全文、`**Example**` 那段两键示例）
- https://nuxt.com/docs/4.x/api/composables/use-runtime-config（"a secret API token that is only accessible on the server"、`runtimeConfig.public` 的定位句、`NUXT_APP_CDN_URL` 那条）
- https://nuxt.com/docs/4.x/directory-structure/env（`.env` 的读取窗口：dev/build/generate 读，跑产物不读）
