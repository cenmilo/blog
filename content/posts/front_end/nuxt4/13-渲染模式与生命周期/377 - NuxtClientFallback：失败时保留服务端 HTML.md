---
url: /posts/aff38ebca34aeafa4e924f9c2b142e45/
title: 377 - NuxtClientFallback：失败时保留服务端 HTML
date: 2027-02-14T11:00:00+08:00
updated: 2027-02-14T11:00:00+08:00
author: cmdragon

summary:
  摘要：`<ClientOnly>` 是无条件跳过服务端；`<NuxtClientFallback>` 相反——它照常做 SSR，**只有当某个子组件在服务端抛错时**才把这块退回到客户端渲染，保住首屏其余内容不被一个坏组件带崩。用前必须开 `experimental.clientFallback`（它还是实验特性）。三个属性带默认值：`fallbackTag`（默认 `div`）、`fallback`、`keepFallback`（默认 `false`）；`@ssr-error` 事件只在服务端触发。一条安全警告很硬：`placeholder`/`fallback` 属性是按 **raw HTML** 渲染的，绝不能把不可信用户输入塞进去（XSS），动态内容改用 `#fallback` 插槽。

categories:
  - 前端开发

tags:
  - nuxt4
  - NuxtClientFallback
  - SSR容错
  - 实验特性
  - XSS
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


376 的 `<ClientOnly>` 一视同仁地把整块留给客户端。但很多时候你**想**服务端渲染这块，只是担心里面某个组件在 SSR 时抛错、把整页带崩。`<NuxtClientFallback>` 就是给这种"容错"场景的：默认正常 SSR，出错才退。

## 它和 ClientOnly 的分工

官方定义点明了触发条件：

> Nuxt provides the `<NuxtClientFallback>` component to render its content on the client if any of its children trigger an error in SSR.

区别就在"什么时候走客户端"：

- `<ClientOnly>`：**总是**不在服务端渲染（376）。
- `<NuxtClientFallback>`：正常参与 SSR，只有**子组件在服务端抛错**时，才把内容退到客户端渲染。

所以它是兜底用的——你预期能服务端出，但防着一个 `BrokeInSSR` 之类的组件把整页搞挂。

## 先用开关打开它

它是实验特性，`::note` 要求先在 `nuxt.config` 里开：

> This component is experimental and in order to use it you must enable the `experimental.clientFallback` option in your `nuxt.config`.

没开 `experimental.clientFallback`，这个组件不可用。

## 三个属性

| 属性 | 类型 | 默认 | 作用（依文档） |
| --- | --- | --- | --- |
| `placeholderTag` \| `fallbackTag` | `string` | `div` | 默认插槽在服务端渲染失败时，用哪个标签作为 fallback 渲染 |
| `placeholder` \| `fallback` | `string` | （无） | 指定插槽渲染失败时要渲染的 fallback 内容 |
| `keepFallback` | `boolean` | `false` | 服务端渲染失败时，保留这个 fallback 内容 |

`keepFallback` 值得单拎：默认 `false` 意味着 fallback 只是过渡；设 `true` 就把"服务端失败时那份 fallback"留下来。

## `@ssr-error` 事件

子组件在 SSR 抛错时会触发 `@ssr-error`，官方特别注明**这个事件只在服务端触发**：

> `@ssr-error`: Event emitted when a child triggers an error in SSR. Note that this will only be triggered on the server.

```vue
<template>
  <NuxtClientFallback @ssr-error="logSomeError">
    <!-- ... -->
  </NuxtClientFallback>
</template>
```

用来在服务端记录"是哪个路由的 SSR 崩了"很合适。

## 一条硬安全警告

官方用 `::warning` 框出，且语气明确：

> The `placeholder` and `fallback` props render content as raw HTML. Do not pass untrusted user input to these props as it may lead to XSS vulnerabilities. Use the `#fallback` or `#placeholder` slots instead for dynamic content that needs proper escaping.

`placeholder`/`fallback` **属性是按原始 HTML 渲染的**——把不可信用户输入放进去会导致 XSS。动态内容要用 `#fallback`/`#placeholder` **插槽**（插槽内容会被正确转义）。这跟 376 的属性用法是同一个坑，但这里文档点名了危险，别为了省事把用户数据拼进 `fallback` 字符串。

```vue
<template>
  <!-- 服务端渲染失败时输出 <span>Hello world</span> -->
  <NuxtClientFallback fallback-tag="span" fallback="Hello world">
    <BrokeInSSR />
  </NuxtClientFallback>
</template>
```

## `#fallback` 插槽

和服务端渲染失败时展示的骨架，用插槽给：

```vue
<template>
  <NuxtClientFallback>
    <!-- ... -->
    <template #fallback>
      <!-- 默认插槽在 SSR 渲染失败时，这段在服务端渲染 -->
      <p>Hello world</p>
    </template>
  </NuxtClientFallback>
</template>
```

## 自查

- NuxtClientFallback 正常 SSR，只在**子组件 SSR 抛错**时才退到客户端渲染——和 ClientOnly 的"总是跳过"相反。
- 用前必须 `experimental.clientFallback: true`（实验特性）。
- 属性 `fallbackTag`（默认 `div`）、`fallback`、`keepFallback`（默认 `false`，设 true 保留 fallback）。
- `@ssr-error` 事件只在服务端触发，用来记哪个路由崩了。
- `placeholder`/`fallback` 属性按 raw HTML 渲染 → 别塞不可信输入（XSS）；动态内容用 `#fallback`/`#placeholder` 插槽。

## 下一篇

下一篇 378 讲延迟水合 `defineLazyHydrationComponent`：服务端照常出 HTML，只是把水合推迟到可见/空闲/交互/媒体查询/条件/定时/永不这七种策略之一，降低首屏交互成本。
