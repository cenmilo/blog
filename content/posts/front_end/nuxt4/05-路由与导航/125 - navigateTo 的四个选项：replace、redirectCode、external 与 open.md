---
url: /posts/bd34d34e40e61e153a55059b50fe0b1f/
title: 125 - navigateTo 的四个选项：replace、redirectCode、external 与 open
date: 2026-11-22T11:00:00+08:00
updated: 2026-11-22T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`navigateTo` 的第二参数是一个 `NavigateToOptions` 对象，官方给了四个键，四种默认值：`replace`（`boolean`、默认 **`false`**）——"**默认情况下，`navigateTo` 会在客户端把给定的路由压入 Vue Router 的实例。这个行为可以通过把 `replace` 设为 `true` 来改变，表示给定的路由应当被替换。**"；`redirectCode`（`number`、默认 **`302`**）——"**`navigateTo` 会重定向到给定路径，并且当重定向发生在服务端时，默认把重定向码设为 `302 Found`。这个默认行为可以通过提供不同的 `redirectCode` 来修改。通常，`301 Moved Permanently` 可以用于永久重定向。**"；`external`（`boolean`、默认 **`false`**）——"**当设为 `true` 时允许导航到一个外部 URL。否则 `navigateTo` 会抛出一个错误，因为外部导航默认是不被允许的。**"；`open`（`OpenOptions`）——"**允许使用 window 的 `open()` 方法来导航到该 URL。这个选项只在客户端适用，在服务端会被忽略。**"。`open` 内部官方又给两项：`target`（`string`、默认 **`'_blank'`**，"**一个不含空格的字符串，指定资源要被加载进的那个浏览上下文的名称**"）与 `windowFeatures`（`OpenWindowFeatures`），后者的七项官方说明分别是：`popup`（"请求一个最小的弹出窗口而不是新标签页，UI 特性由浏览器决定"）、`width` 或 `innerWidth`（"指定内容区的宽度（**最小 100 像素**），包含滚动条"）、`height` 或 `innerHeight`（同样最小 100 像素）、`left` 或 `screenX`（"设置新窗口相对于屏幕左边缘的水平位置"）、`top` 或 `screenY`（垂直位置）、**`noopener`（"阻止新窗口通过 `window.opener` 访问发起它的窗口"）**、**`noreferrer`（"阻止 Referer 头被发送，并隐式启用 `noopener`"）**。这一组在类型上还有一条硬约束——官方签名把它写成 `XOR<{ width?: number }, { innerWidth?: number }>` 这种互斥组合，也就是**成对的两个键只能给一个**。另外 `external` 这一项在 `<NuxtLink>` 上也有同名 prop，但用途不同：链接侧官方用它来"绕过 Vue Router 的内部路由机制"，指向 `/public` 里的静态文件或同一域名下的另一个应用。

categories:
  - 前端开发

tags:
  - nuxt4
  - navigateTo
  - 重定向
  - open
  - 外部链接

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`navigateTo('/x')` 谁都会写。但"这次要不要留在历史里""服务端返回 302 还是 301""能不能跳出去""要不要开新窗口"这四件事，全在第二参数那四个键上。

## 四个键的默认值先记住

```ts
interface NavigateToOptions {
  replace?: boolean
  redirectCode?: number
  external?: boolean
  open?: OpenOptions
}
```

| 键 | 类型 | 默认 | 管什么 |
| --- | --- | --- | --- |
| `replace` | `boolean` | **`false`** | 压入历史条目，还是替换当前条目 |
| `redirectCode` | `number` | **`302`** | **服务端**重定向时用的状态码 |
| `external` | `boolean` | **`false`** | 允不允许跳出到外部 URL |
| `open` | `OpenOptions` | —— | 用 `window.open()` 打开，**只在客户端生效** |

四个默认值连起来就是官方默认行为：**在客户端压入历史、在服务端发 302、不许跳外部、不开新窗口。**

## `replace`：历史栈里留不留这一格

官方说明：

> 默认情况下，`navigateTo` 会在客户端把给定的路由**压入** Vue Router 的实例。这个行为可以通过把 `replace` 设为 `true` 来改变，表示给定的路由应当**被替换**。

这与 123 里 `router.replace()` 的定义是同一件事（"通过替换历史栈里的当前条目来程序化地导航"）。

| 场景 | `replace` |
| --- | --- |
| 正常的前进式跳转（去详情、去结账） | 默认 `false`，用户能后退 |
| 登录成功跳首页、废弃 URL 改写 | `true`，**别让"后退"回到登录前那一格** |

## `redirectCode`：只有服务端那次才有意义

官方说明：

> `navigateTo` 会重定向到给定路径，并且**当重定向发生在服务端时**，默认把重定向码设为 `302 Found`。这个默认行为可以通过提供不同的 `redirectCode` 来修改。通常，`301 Moved Permanently` 可以用于永久重定向。

注意措辞里的限定："当重定向发生在服务端时"。**浏览器内导航没有 HTTP 状态码**，所以这一格只在服务端渲染阶段起作用。

124 里官方的中间件示例正是用它做永久跳转：

```ts
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.path !== '/search') {
    // setting the redirect code to '301 Moved Permanently'
    return navigateTo('/search', { redirectCode: 301 })
  }
})
```

| 码 | 官方语气 | 什么时候用 |
| --- | --- | --- |
| `302` | 默认 | 目标还会变、或带条件（未登录去登录页） |
| `301` | "**通常**，301 Moved Permanently 可以用于永久重定向" | 老 URL 彻底搬家，希望缓存与搜索引擎记住 |

## `external`：默认是"抛错"，不是"忽略"

官方把这格的两种情形列成一个四象限：

| | **内部 URL** | **外部 URL** |
| --- | --- | --- |
| **不带 `external: true`** | 正常导航 | **抛出一个错误** |
| **带 `external: true`** | 以**整页刷新**方式导航 | 正常导航 |

配套说明：

> 当设为 `true` 时允许导航到一个外部 URL。否则 `navigateTo` 会抛出一个错误，**因为外部导航默认是不被允许的**。

官方的示例把这两格直接写在一起：

```vue
<script setup lang="ts">
// will throw an error;
// navigating to an external URL is not allowed by default
await navigateTo('https://nuxt.com')

// will redirect successfully with the 'external' parameter set to 'true'
await navigateTo('https://nuxt.com', {
  external: true,
})
</script>
```

**这一格里两个方向都要小心**：

- 忘了加 `external: true` → 跳外部地址**抛错**（不是静默失败）。
- 给内部地址加了 `external: true` → 能跳，但变成**整页刷新**，把 SPA 的好处丢了（水合重来、状态丢失）。

## `open`：新窗口，且只在客户端

官方说明：

> 允许使用 window 的 `open()` 方法来导航到该 URL。**这个选项只在客户端适用，在服务端会被忽略。**

官方的完整用法：

```vue
<script setup lang="ts">
// will open 'https://nuxt.com' in a new tab
await navigateTo('https://nuxt.com', {
  open: {
    target: '_blank',
    windowFeatures: {
      width: 500,
      height: 500,
    },
  },
})
</script>
```

### 两项配置

| 项 | 官方说明 | 默认 |
| --- | --- | --- |
| `target` | **一个不含空格的字符串**，指定资源要被加载进的那个**浏览上下文的名称** | **`'_blank'`** |
| `windowFeatures` | 一个对象，接受下列属性 | —— |

`windowFeatures` 官方列了七项：

| 属性 | 官方说明 |
| --- | --- |
| `popup` | 请求一个最小的弹出窗口而不是新标签页，UI 特性由浏览器决定 |
| `width` 或 `innerWidth` | 指定**内容区**的宽度（**最小 100 像素**），包含滚动条 |
| `height` 或 `innerHeight` | 指定内容区的高度（最小 100 像素），包含滚动条 |
| `left` 或 `screenX` | 设置新窗口相对于屏幕左边缘的水平位置 |
| `top` 或 `screenY` | 设置新窗口相对于屏幕顶边缘的垂直位置 |
| `noopener` | **阻止新窗口通过 `window.opener` 访问发起它的窗口** |
| `noreferrer` | **阻止 Referer 头被发送，并隐式启用 `noopener`** |

### 类型上的那条互斥

官方的类型定义写的是：

```ts
type OpenWindowFeatures = {
  popup?: boolean
  noopener?: boolean
  noreferrer?: boolean
} & XOR<{ width?: number }, { innerWidth?: number }>
  & XOR<{ height?: number }, { innerHeight?: number }>
  & XOR<{ left?: number }, { screenX?: number }>
  & XOR<{ top?: number }, { screenY?: number }>
```

`XOR` 的含义是**二选一**：`width` 与 `innerWidth` 这一对只能给其中一个（高度、左右、上下同理）。这与表格里"或"的写法互相印证，也是类型检查会拦住的地方。

## 别和 `<NuxtLink>` 的同名 prop 混起来

`external` 这个词在链接组件上也存在，但官方给它的任务不一样。链接侧的说法是：

> 默认情况下，`<NuxtLink>` 对相对路由使用 Vue Router 的客户端导航。**当链接指向 `/public` 目录里的静态文件，或指向同一域名上托管的另一个应用时，可能会造成意外的 404**，因为它们不属于客户端路由。**这种情况下你可以对 `<NuxtLink>` 使用 `external` prop 来绕过 Vue Router 的内部路由机制。**官方又说：`external` prop 明确表明该链接是外部的，`<NuxtLink>` 会把它渲染成一个标准的 HTML `<a>` 标签。

两个官方示例：

```vue
<NuxtLink to="/example-report.pdf" external>Download Report</NuxtLink>
<NuxtLink to="/another-app" external>Go to Another App</NuxtLink>
```

对照表：

| | `navigateTo(..., { external: true })` | `<NuxtLink external>` |
| --- | --- | --- |
| 触发方式 | 代码里执行 | 用户点击 |
| 主要用途 | 允许跳出到外部站点 | 让相对路径绕过路由表，指向 `/public` 文件**或同域另一个应用** |
| 内部地址的后果 | 整页刷新 | 渲染成普通 `<a>`，交给服务器处理 |

还有一条链接侧的便利，官方写成了 tip：**"当你把一个对象传给 `to` prop 时，`<NuxtLink>` 会继承 Vue Router 对查询参数的处理。键与值会被自动编码，所以你不需要手动调用 `encodeURI` 或 `encodeURIComponent`。"**

## 自查

1. 四个 option 的默认值分别是什么？把这四个默认值连起来读，官方默认行为是什么？
2. `redirectCode` 为什么"只在服务端那次才有意义"？301 官方用什么语气推荐？
3. `external` 不设时跳外部 URL 会怎样？给内部 URL 加 `external: true` 会付出什么代价？
4. `open` 在服务端会发生什么？`target` 的默认值是什么？
5. `windowFeatures` 里为什么 `width` 与 `innerWidth` 不能同时给？官方类型里用什么表达这件事？
6. `noopener` 与 `noreferrer` 各挡掉什么？后者还顺带做了什么？
7. `navigateTo` 的 `external` 与 `<NuxtLink>` 的 `external` 分别解决什么问题？

## 下一篇

126 把跳转的安全面收齐：外部地址默认被拒、`noopener`/`noreferrer`、`<NuxtLink>` 给绝对链接与 `target` 链接自动加 `rel` 的默认值，以及 `rel`/`noRel` 为什么不能同时用。
