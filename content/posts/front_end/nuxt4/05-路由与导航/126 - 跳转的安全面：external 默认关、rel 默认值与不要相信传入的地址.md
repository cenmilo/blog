---
url: /posts/9ea4da62e6e817f504d854f2680d06b0/
title: 126 - 跳转的安全面：external 默认关、rel 默认值与不要相信传入的地址
date: 2026-11-23T09:00:00+08:00
updated: 2026-11-23T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：跳转这件事，官方已经把三道默认防线写进了文档，剩下的要靠自己的校验。**第一道在 `navigateTo`：外部导航默认不被允许**——官方的说明是"**当设为 `true` 时允许导航到一个外部 URL。否则 `navigateTo` 会抛出一个错误，因为外部导航默认是不被允许的。**"，四象限表里"不带 `external: true` + 外部 URL = 抛出一个错误"就是这一道门；而它还有一个代价方向：**带上 `external: true` 之后，内部 URL 会以整页刷新方式导航**。**第二道在 `<NuxtLink>` 的 `rel` 默认值**，官方原文：**"一个值为 `noopener noreferrer` 的 `rel` 属性会被默认应用于带 `target` 属性的链接，或绝对链接（例如以 `http://`、`https://` 或 `//` 开头的链接）。"** 官方给的理由是两条：**"`noopener` 解决了老浏览器里的一个安全缺陷"**、**"`noreferrer` 通过不向被链接的站点发送 `Referer` 头来为你的用户改善隐私"**，并且补了一句 **"这些默认值对 SEO 没有负面影响，被认为是最佳实践。"** 需要改写时用 `rel` 或 `noRel` 两个 prop，官方同时挂了一条 note：**"`noRel` 与 `rel` 不能一起使用。`rel` 会被忽略。**"**第三道在 `open.windowFeatures`**：`noopener`"**阻止新窗口通过 `window.opener` 访问发起它的窗口**"，`noreferrer`"**阻止 Referer 头被发送并隐式启用 `noopener`**"。注意官方那个默认适用范围里包含"以 `//` 开头的链接"——**协议相对链接被算作绝对链接**，这正是"只判断地址是不是以 `/` 开头"不够用的原因。剩下要自己做的是**来源校验**：`navigateTo` 的 `to` 类型是 `RouteLocationRaw | undefined | null`、默认 `'/'`，外部默认被拒，但**同域内的任意路径它照样会跳**，所以把 `route.query.redirect` 之类用户可控的值直接交给 `navigateTo` 是不安全的。还有两条边界要记得：`navigateTo` **不能在 Nitro 路由里使用**（服务端重定向改用 `sendRedirect`），而**路由中间件不会为 `/api/*` 或其他服务端请求运行**——这两条合起来意味着"前端跳转发保护不到接口"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 安全
  - 跳转
  - rel
  - 外部链接

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


"登录后跳回原来那页"这种需求，是新手最容易写出漏洞的地方。官方其实已经把三道默认防线写好了，先弄懂它们，再补上自己那一段校验。

## 第一道：外部地址默认跳不过去

`navigateTo` 对 URL 的处理，官方给的是四象限：

| | 内部 URL | 外部 URL |
| --- | --- | --- |
| 不带 `external: true` | 正常导航 | **抛出一个错误** |
| 带 `external: true` | 以**整页刷新**方式导航 | 正常导航 |

配套说明："当设为 `true` 时允许导航到一个外部 URL。否则 `navigateTo` 会抛出一个错误，**因为外部导航默认是不被允许的**。"

这道门的价值在于**默认拒绝**：把用户传进来的地址直接丢给 `navigateTo`，最坏情况是抛错，而不是把用户送到别人的站。官方的例子把这点写得很直白：

```ts
// will throw an error;
// navigating to an external URL is not allowed by default
await navigateTo('https://nuxt.com')

// will redirect successfully with the 'external' parameter set to 'true'
await navigateTo('https://nuxt.com', { external: true })
```

但注意另一侧的代价：**一旦你为了让某个地址通过而全局加 `external: true`，所有内部跳转都会退化成整页刷新**——那等于把 SPA 的水合与状态全丢掉。所以这一格只该在"确实要出到别的站点"时打开。

## 第二道：`<NuxtLink>` 自动给外部链接加 `rel`

这一条是最容易被忽略、又最实在的默认防线。官方原文：

> **一个值为 `noopener noreferrer` 的 `rel` 属性会被默认应用于带 `target` 属性的链接，或绝对链接（例如以 `http://`、`https://` 或 `//` 开头的链接）。**
>
> - `noopener` 解决了老浏览器里的一个安全缺陷。
> - `noreferrer` 通过不向被链接的站点发送 `Referer` 头来为你的用户改善隐私。
>
> 这些默认值对 SEO 没有负面影响，被认为是最佳实践。

官方的对照示例（注释就是渲染结果）：

```vue [app/app.vue]
<template>
  <NuxtLink to="https://twitter.com/nuxt_js">
    Nuxt Twitter
  </NuxtLink>
  <!-- <a href="https://twitter.com/nuxt_js" rel="noopener noreferrer">...</a> -->

  <NuxtLink
    to="https://discord.nuxtjs.org"
    rel="noopener"
  >
    Nuxt Discord
  </NuxtLink>
  <!-- <a href="https://discord.nuxtjs.org" rel="noopener">...</a> -->

  <NuxtLink to="/about" target="_blank">About page</NuxtLink>
  <!-- <a href="/about" target="_blank" rel="noopener noreferrer">...</a> -->
</template>
```

这段示例里有三点要读出来：

| 观察 | 结论 |
| --- | --- |
| 绝对链接自动带 `rel="noopener noreferrer"` | 默认防线，不需要你写 |
| 显式写 `rel="noopener"` 会**替换**默认值 | 覆盖方式是 `rel` prop |
| **内部地址 `/about` 只要加了 `target="_blank"`，也会被加上 `rel="noopener noreferrer"`** | 触发条件是"新浏览上下文"，不是"跨域" |

第三条最实用：**"新窗口打开"这个动作本身就足以触发这道防线**，同域也一样。

想彻底不要 `rel`，官方给了 `noRel`：

```vue [app/app.vue]
<template>
  <NuxtLink to="https://github.com/nuxt" no-rel>
    Nuxt GitHub
  </NuxtLink>
  <!-- <a href="https://github.com/nuxt">...</a> -->
</template>
```

并挂了一条限制：**"`noRel` 与 `rel` 不能一起使用。`rel` 会被忽略。"**

### `//` 这个细节为什么重要

官方那句适用范围里明确列了三种开头：`http://`、`https://`、**`//`**。也就是说 **协议相对链接在 Nuxt 眼里就是绝对链接**（它指向别的服务器，只是省略了协议）。

这正是"只判 `startsWith('/')` 就放行"这种写法不安全的原因：`//evil.example/x` 也以 `/` 开头，但它是外部地址——从 `navigateTo` 的默认拒绝角度看，它属于外部 URL；从链接角度看，它已经被算作绝对链接。

## 第三道：`window.open` 的两个特征位

用 `open` 打开新窗口时，官方在 `windowFeatures` 里给了同样两个开关：

| 属性 | 官方说明 |
| --- | --- |
| `noopener` | **阻止新窗口通过 `window.opener` 访问发起它的窗口** |
| `noreferrer` | **阻止 Referer 头被发送，并隐式启用 `noopener`** |

```vue
<script setup lang="ts">
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

**这条与第二道是同一件事的两个入口**：链接侧靠默认 `rel`，代码侧靠 `windowFeatures`。`noreferrer` 隐式启用 `noopener` 这一点也说明：需要挡 `window.opener` 时，给 `noreferrer` 就够了。

## 剩下的要靠你：来源校验

`navigateTo` 的 `to` 类型是 `RouteLocationRaw | undefined | null`，官方给了默认值"**当传入 `undefined` 或 `null` 时，它默认到 `'/'`**"。这两条能挡住"目标为空"，但**挡不住"目标是一个合法的同域路径、只是不该去"**：默认防线只管内外，不管同域里哪一条。

所以下面这种写法要避免：

```ts
// ❌ 直接把用户可控的查询参数交给 navigateTo
await navigateTo(route.query.redirect as string)
```

三条可选做法，按严格程度递增：

| 做法 | 说明 |
| --- | --- |
| **只允许 `name` + `params`** | 官方给的 `to` 对象形态之一就是 `{ name: 'product', params: { id: 1 } }`（124）。名字是你自己路由表里的，用户没法借它跳去别处 |
| **白名单前缀校验** | 只放行你确定的业务前缀（`/posts/`、`/account/`），其余一律回退到 `'/'` |
| **自己判绝对形态** | 至少把 `//` 开头（协议相对）与含协议头的地址都排除——官方那三种开头就是判据清单 |

另外一条隐私向的规则：跳转地址上的查询参数会跟着走。官方在岛屿那一节写得更直白——查询参数"**可能出现在服务器访问日志、CDN 缓存与 HTTP `Referer` 头中**"（107）。**令牌、邮箱这类值别放进跳转 URL 的 query。**

## 两条"保护不到"的边界

| 官方条款 | 影响 |
| --- | --- |
| "`navigateTo` 不能在 Nitro 路由里使用。要在 Nitro 路由里做服务端重定向，请改用 `sendRedirect`。" | 接口层要跳转/拦截，不归 `navigateTo` 管 |
| "路由中间件**不会**为服务端路由（例如 `/api/*`）或其他服务端请求运行。要对这些请求应用中间件，请使用服务端中间件。"（110 引过） | 前端中间件里的登录判断**保护不到接口**——接口要自己判权限 |

**结论：跳转层面的校验只能保护"页面"，接口必须各自把关。**

## 六条自查清单

1. 有 `external: true` 的地方，是不是真的在跳外部站点？（内部地址会整页刷新）
2. 有没有把 `route.query` / 表单里的地址直接交给 `navigateTo`？
3. 外部地址的判据用全了吗？（`http://`、`https://`、`//` 三种开头）
4. 用 `target="_blank"` 或 `open` 新窗口时，`noopener`/`noreferrer` 是否还成立（有没有被 `rel` 或 `noRel` 关掉）？
5. 跳转 URL 的 query 里有没有敏感值？
6. 接口的权限判断是不是只写在了前端中间件里？

## 自查

1. 官方的第一道防线是什么？它的默认行为是"忽略"还是"抛错"？
2. `<NuxtLink>` 的默认 `rel` 应用于哪两类链接？官方给的两种理由分别是什么？
3. 一个同域内部链接，什么情况下也会被加上 `rel="noopener noreferrer"`？
4. `rel` 与 `noRel` 同时写会怎样？官方怎么说？
5. 为什么"只判断地址以 `/` 开头"是不够的？官方哪句话给了依据？
6. `navigateTo` 的哪两条边界意味着"跳转校验保护不到接口"？

## 下一篇

127 讲 `abortNavigation`：它只能用在路由中间件处理器里、签名返回 `false`，以及不带参数、带字符串、带 Error 对象三种写法分别对应什么场景。
