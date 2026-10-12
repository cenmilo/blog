---
url: /posts/3d78e5b696ac7b4d918fdea678122c7a/
title: 类型化 runtimeConfig：让 TS 认得你的键
date: 2027-02-03T11:00:00+08:00
updated: 2027-02-03T11:00:00+08:00
author: cmdragon

summary:
  摘要：runtimeConfig 默认会自动生成类型——Nuxt 用 unjs/untyped 从你在 nuxt.config 写的键推出接口，所以多数时候你 useRuntimeConfig().myKey 直接就有类型。当自动推出来的不对、或你想显式约束时，官方给的是手动 declare 两个接口：RuntimeConfig（私有层）与 PublicRuntimeConfig（public 层），都写在 declare module 'nuxt/schema' 里、末尾带 export {}。这一篇把自动与手动两条讲清，补一句模块作者该扩 @nuxt/schema 而非 nuxt/schema，并接回 289 的"增补要放对类型上下文"。

categories:
  - 前端开发

tags:
  - nuxt4
  - runtimeConfig
  - TypeScript
  - 类型增补
  - nuxt-schema
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（343）让 runtimeConfig 的"值"在运行时正确落地。这一篇管它的"类型"——`useRuntimeConfig()` 返回的东西 TS 认不认识你的键。

## 默认：自动推类型

runtime-config 页 "Typing Runtime Config" 第一句：

> Nuxt tries to automatically generate a typescript interface from provided runtime config using unjs/untyped.

意思是：你在 `nuxt.config` 的 `runtimeConfig` 里写了 `apiSecret`、`public.apiBase`，Nuxt 会用 untyped 从这些默认值**自动推出接口**。所以 `useRuntimeConfig().apiSecret`、`useRuntimeConfig().public.apiBase` 通常开箱就有类型，你不用手写。

这也是为什么 343 强调"默认值要在 nuxt.config 里声明"——声明过不只是覆盖机制需要，类型也是从这份声明来的。

## 手动：declare 两个接口

自动推的不满意（比如想让某键必填、或类型更精确），官方给的手动写法：

```ts [index.d.ts]
declare module 'nuxt/schema' {
  interface RuntimeConfig {
    apiSecret: string
  }
  interface PublicRuntimeConfig {
    apiBase: string
  }
}
// It is always important to ensure you import/export something when augmenting a type
export {}
```

两个接口对应 340 讲的两层：

- `RuntimeConfig` → 私有层（顶层键）。
- `PublicRuntimeConfig` → `public` 层的形状。

`declare module 'nuxt/schema'` 是在给这个模块**扩展**接口。末尾 `export {}` 那句被官方在注释里专门强调——augment 文件必须含 import/export，否则是全局脚本、`declare module` 语义会变成覆盖而不是扩展（289 讲类型化 provide 时说过同一条）。

## nuxt/schema 与 @nuxt/schema：谁扩哪个

runtime-config 页有条 note，模块作者尤其要看：

> `nuxt/schema` is provided as a convenience for end-users to access the version of the schema used by Nuxt in their project. Module authors should instead augment `@nuxt/schema`.

- **应用作者（在自己项目里扩类型）**：用 `declare module 'nuxt/schema'`——它是 Nuxt 给你项目用的那套 schema。
- **模块作者（写发布的模块）**：扩 `@nuxt/schema`，别绑死某个项目的 `nuxt/schema`。

写错目标，模块发布出去可能对不上消费者项目的 schema 版本。

## 放对类型上下文（接 289）

runtimeConfig 的类型是**两端都要用的**（服务端读私有、客户端读 public）。typescript 页 "Augmenting Types with Project References" 的规则：增补文件要放在能扫到它的类型上下文目录里；跨 app/server 共享的放 `shared/`。所以这个 `index.d.ts` 通常放 `shared/`（或 app 与 server 都覆盖得到的位置），否则可能"声明了但某个上下文看不到"。

跑过 `nuxt dev`/`build`/`prepare` 之后类型才刷新；刚加完 .d.ts 没生效，先重跑 `nuxt prepare`（289 同一条）。

## 自查

- 多数时候 runtimeConfig 类型自动推（untyped），键在 nuxt.config 声明了就有类型。
- 要手动约束：declare `RuntimeConfig`（私有层）+ `PublicRuntimeConfig`（public 层），写在 `declare module 'nuxt/schema'` 里、带 `export {}`。
- 模块作者扩 `@nuxt/schema`，应用作者扩 `nuxt/schema`，别混。
- 增补文件放对类型上下文目录（跨端共享放 `shared/`）；没生效先重跑 `nuxt prepare`。

## 下一篇

下一篇 345 把配置来源排个优先级全景：nuxt.config、项目 .nuxtrc、家目录 .nuxtrc、以及运行时环境变量各管哪一层、谁盖谁。

## 本文依据

- Runtime Config（`guide/going-further/runtime-config`）："Typing Runtime Config" 节（"Nuxt tries to automatically generate a typescript interface from provided runtime config using unjs/untyped."、手动 declare `RuntimeConfig`/`PublicRuntimeConfig` 于 `declare module 'nuxt/schema'` + `export {}`、note "`nuxt/schema` is provided as a convenience ... Module authors should instead augment `@nuxt/schema`"）。
- TypeScript（`guide/concepts/typescript`）：增补要放对类型上下文目录（app/server/shared）、生成时机 dev/build/prepare（见 289/344）。
