---
url: /posts/b46d653e6e78fb977196f8b366213d1a/
title: future 节：提前启用未来默认行为的开关
date: 2027-02-06T10:00:00+08:00
updated: 2027-02-06T10:00:00+08:00
author: cmdragon

summary:
  摘要：nuxt.config 有个 future 命名空间，专门给"以后会变成默认、现在想提前打开"的行为。这一篇讲它目前的三个键：compatibilityVersion（设 5 就把整套默认切到 Nuxt 5 行为，含启用 Vite Environment API，用来提前测 Nuxt 5）、multiApp（提前开实验性的多应用支持）、typescriptBundlerResolution（把 TS 模块解析切成 Bundler 模式，对 Nuxt/Vite 这类框架是推荐值，设 false 退回旧的 Node 模式）。和 compatibilityDate（351 锁默认防漂移）不同，future 是"主动往前跨一步"。

categories:
  - 前端开发

tags:
  - nuxt4
  - future
  - compatibilityVersion
  - typescript
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（351）的 compatibilityDate 是"把默认锁在现在"。这一篇的 `future` 相反——"把某些未来的默认提前打开"。

## future 是干嘛的

nuxt-config 的 `future` 节只有一句 + 一个指向：

> There is also a `future` namespace for early opting-in to new features that will become default in a future (possibly major) version of the framework.

一句话：**给"以后会成为默认、现在想先尝"的行为开一个提前量。** 形状是：

```ts
export default defineNuxtConfig({
  future: {
    // 这些键
  },
})
```

## 键一：compatibilityVersion

官方 features 页的说明：

> This enables early access to Nuxt features or flags. Setting `compatibilityVersion` to `5` changes defaults throughout your Nuxt configuration to opt in to Nuxt v5 behaviour, including enabling the Vite Environment API.

```ts
export default defineNuxtConfig({
  future: {
    compatibilityVersion: 5,
  },
})
```

要点：

- 设成 `5` 会把**整套默认**切到 Nuxt 5 行为，包括打开 Vite Environment API。
- 它是"提前体验下一个大版本"，官方把它和"测试 Nuxt 5"关联（read-more 指向 upgrade#testing-nuxt-5）。
- 这也解释了前面很多篇见过的"compatibilityVersion: 5 时某默认变另一样"（比如 payloadExtraction 默认 `'client'`、normalizePageNames、extractSerializablePageMeta 要 cv5 等）——就是这个开关把它们提前生效了。

和 351 的 compatibilityDate 区别再钉一次：compatibilityDate 是**日期**、锁"当前这套默认别漂移"；compatibilityVersion 是**版本号**、主动切到下一大版本的默认。

## 键二：multiApp

> This enables early access to the experimental multi-app support. You can follow the tracker issue #21635 to see the progress of multi-app support in Nuxt.

```ts
future: { multiApp: true }
```

提前开"多应用支持"这个实验特性，官方挂了一个 GitHub tracker issue（#21635）让你跟进度。它是实验性的——除非你真需要在一个项目里跑多个 Nuxt 应用，一般别开。

## 键三：typescriptBundlerResolution

官方说明：

> This enables 'Bundler' module resolution mode for TypeScript, which is the recommended setting for frameworks like Nuxt and Vite. It improves type support when using modern libraries with `exports`. ... You can set it to false to use the legacy 'Node' mode, which is the default for TypeScript.

对 Nuxt/Vite 这类框架，Bundler 解析模式是**推荐**的（现代库用 `exports` 字段时类型支持更好）。设 `false` 退回 TS 传统默认 Node 模式。这条对刚上手的人通常是"不用动、让它按推荐走"，知道有这么个开关、遇到某类库的类型解析问题时可回来看它。

## future 与 experimental 的分工

都叫"提前开"，但定位不同：

- `future.*`：**未来会变成默认**的行为，现在提前打开（打开后基本是"往正确方向走"）。
- `experimental.*`：**还在实验、可能改也可能删**（263/267 那些，官方明说可能移除）。

一句话：future 是"迟早的、我先上"，experimental 是"不保证的、我尝鲜"。

## 自查

- 想提前切到 Nuxt 5 的整套默认 → `future.compatibilityVersion: 5`（也开 Vite Environment API）；它和 compatibilityDate（锁当前默认）方向相反，别混。
- multiApp、typescriptBundlerResolution 按需开；前者实验性一般别动，后者一般保持推荐值不用改。
- future 是"未来会变默认"、experimental 是"不保证会留"，两个命名空间别当成一个。

## 下一篇

P12 配置与环境变量继续按索引推进（353 起）。编号连续，日期从下一篇 +30 分钟续。

## 本文依据

- nuxt-config future 节（`api/nuxt-config`）：read-more 指向 features#features、"opting-in to new features that will become default in a future (possibly major) version"。
- Features `future` 节（`guide/going-further/features`）：定位句 "There is also a `future` namespace for early opting-in to new features that will become default in a future (possibly major) version of the framework."；compatibilityVersion（"Setting compatibilityVersion to 5 changes defaults throughout ... opt in to Nuxt v5 behaviour, including enabling the Vite Environment API"、read-more 指向 upgrade#testing-nuxt-5）；multiApp（"early access to the experimental multi-app support"、tracker #21635）；typescriptBundlerResolution（"Bundler module resolution ... recommended setting for frameworks like Nuxt and Vite"、false 退回 legacy Node 模式）。
- compatibilityDate（`api/nuxt-config` / `errors/b5001`）：锁当前默认、防漂移（见 351）。
