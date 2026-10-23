---
url: /posts/563b713fbb9db646eb0652229a56ebd5/
title: 517 - buildId 与过期构建检测：manifest 撑起来的两件事
date: 2027-04-02T10:00:00+08:00
updated: 2027-04-02T10:00:00+08:00
author: cmdragon

summary:
  摘要：`buildId` 的官方定义只有一句："A unique identifier matching the build. This may contain the hash of the current state of the project."，类型 `string`，文档给的默认值是一串 UUID 形状的样例（`4a2e2d30-418f-41df-8e58-ed5df06de7fd`）。它和 `baseURL`、`buildAssetsDir`、`cdnURL` 一起住在 `runtimeConfig.app` 这个**保留命名空间**里。真正决定"客户端知不知道线上换了新版本"的不是它，而是 `experimental.appManifest`——manifest 支撑的两项能力写在错误码 E5001 那一节：**客户端的路由规则匹配**与**预渲染载荷探测**。`checkOutdatedBuildInterval`（检查新构建的时间间隔）在 `appManifest` 为 `false` 时**跟着一起被关掉**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建
  - 缓存
  - manifest
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上线新版本以后，用户的标签页里还跑着旧 JS，点到某些页面就报错。这类问题在 Nuxt 里有专门的机制管，但入口很隐蔽——它挂在一个实验性开关上。

## buildId 是什么

`api/nuxt-config` 对 `buildId` 的全部记载是一句：

> A unique identifier matching the build. This may contain the hash of the current state of the project.

- **类型**：`string`
- **默认**：`"4a2e2d30-418f-41df-8e58-ed5df06de7fd"`

三处措辞要按字面读。第一，它**匹配一次构建**，不是匹配版本、不是匹配部署。第二，"This **may** contain the hash of the current state of the project"——官方用的是 may，也就是说**"它一定等于项目内容哈希"是过度解读**。第三，文档给的默认值是一个写死的 UUID 样例，这行更像格式示意而不是"你的构建会拿到这个值"。

**官方在这一节没有写这个值如何生成、会不会每次构建都变。** 要确认只能自己取证：连续跑两次 `npx nuxt build`，在产物里搜这个标识看它变没变。

它出现的另一处是 `runtimeConfig` 的默认值 JSON：

```json
{
  "public": {},
  "app": {
    "buildId": "4a2e2d30-418f-41df-8e58-ed5df06de7fd",
    "baseURL": "/",
    "buildAssetsDir": "/_nuxt/",
    "cdnURL": ""
  },
  "nitro": {
    "envPrefix": "NUXT_"
  }
}
```

这四项（`buildId`/`baseURL`/`buildAssetsDir`/`cdnURL`）在 `runtimeConfig.app` 下是**保留命名空间**——错误码 `NUXT_B5003` 讲的就是把值直接写进 `runtimeConfig` 顶层结果需要挪到 `public`（客户端可见）或顶层（仅服务端）这一类问题。也就是说：这几个键由框架填，不该你手写去覆盖它做业务用途。

## 真正管"新构建"的是 manifest

`experimental.appManifest` 那节只有一句定位加一句状态：**"Use app manifests to respect route rules on client-side."**，以及 "This flag is enabled by default, but you can disable this feature"（关闭示例是给 `appManifest: false`）。

它到底支撑了什么，写在错误码 `NUXT_E5001` 那一节。它的触发条件与影响面一起给了：

> Code that relies on the app manifest ran while `experimental.appManifest` is disabled. The manifest powers features such as route rules matching and prerendered-payload detection on the client, so it must be enabled for them to work.

拆出来是两项能力，都在**客户端**：**路由规则匹配**与**预渲染载荷探测**。修法就是重新打开：

```ts
export default defineNuxtConfig({
  experimental: {
    appManifest: true,
  },
})
```

对照前面那节你会看到一处口径差：`appManifest` 那节说"默认开启"，而 E5001 的前提是这个开关被关掉。**这两处并不矛盾**（一个是常态、一个是被关闭后的后果），但要注意关它的代价不止"不尊重客户端路由规则"——**预渲染载荷探测也一起没了**，而这正是静态托管站点会踩的那条。

## 过期构建检查挂在它下面

`checkOutdatedBuildInterval` 那节的三行值得逐字看：

> Set the time interval (in ms) to check for new builds. Disabled when `experimental.appManifest` is `false`.
>
> Set to `false` to disable.

示例：

```ts twoslash [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    checkOutdatedBuildInterval: 3600000, // 1 hour, or false to disable
  },
})
```

三点：单位是**毫秒**；这个间隔检查的是**有没有新构建**；**依赖关系是单向的**——`appManifest: false` 会让这项一起失效，但反过来关掉这项不影响 manifest 本身。

另外这一节**没有列 Type 与 Default**，只给了一个带注释的示例值 `3600000`。所以"默认就是一小时"这种说法在文档里找不到依据，能确证的只有示例用的这个数与注释里"1 hour, or false to disable"这句。

## 长会话站点的实际形状

把三块拼起来，能给出一个有依据的判断链：**如果你的站点会被人长时间开着（后台系统、大屏、编辑器类页面），关掉 `appManifest` 等于同时关掉客户端路由规则匹配、预渲染载荷探测和过期构建检查。** 想保留"发现新构建"这件事，`experimental.appManifest` 必须开着，然后按你的容忍度决定 `checkOutdatedBuildInterval` 用示例值、改小还是设 `false`。

改完之后**"用户被提示刷新"这一步官方没有记载**——我在全套 4.x 文档里没找到"发现新构建后如何提示用户"的章节，所以别按其他框架的经验假设有个默认弹窗。要确认行为，最短路径是发一次新版本、在旧标签页里等一个间隔，看控制台与界面实际发生了什么。

## 自查

- `buildId` 是 `string`，官方只说"匹配一次构建"且 **may** contain 项目状态哈希；生成规则与是否每次变化文档未写。
- 它与 `baseURL`/`buildAssetsDir`/`cdnURL` 同在 `runtimeConfig.app` 这个保留命名空间下（B5003 讲的就是这类放置问题）。
- 客户端的路由规则匹配与预渲染载荷探测由 app manifest 支撑（E5001 那句 "the manifest powers features such as ..."），这个开关默认开。
- `checkOutdatedBuildInterval` 单位毫秒、`appManifest` 为 `false` 时它一起失效；该节没列默认值，只有 `3600000` 的示例。
- 「发现新构建后怎么提示用户」在 4.x 文档里没有记载。

## 下一篇

下一篇 518 讲环境变量：`.env` 只在 dev/build/generate 三个时机被读、生产**不读你的 `.env`**，以及 `NUXT_` 前缀这条硬要求。
