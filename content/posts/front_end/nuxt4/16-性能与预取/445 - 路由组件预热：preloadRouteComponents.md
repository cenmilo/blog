---
url: /posts/44a9409d9267371c2812e6bcaf3cb6fd/
title: 445 - 路由组件预热：preloadRouteComponents
date: 2027-03-09T10:00:00+08:00
updated: 2027-03-09T10:00:00+08:00
author: cmdragon

summary:
  摘要：`preloadRouteComponents` 预热的是**一条路由的组件**，参数是路由路径，用在"不是点链接到达"的跳转前面。官方给它的定位有两处特别值得照抄：一是那句提示——"如果你用 `NuxtLink`，Nuxt 已经自动预取需要的路由了"，所以这个函数只补 `navigateTo`、表单提交后跳转这类路径；二是示例里那行注释——**故意不 await**，因为 await 会挡住当前组件的 setup。和 444 那两个组件级函数一样，它在服务端没有效果。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预热
  - preloadRouteComponents
  - 路由
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

登录页点"提交"，请求成功之后 `navigateTo('/dashboard')`——页面白了一下才出内容。

原因不神秘：dashboard 那个页面文件的 chunk 此刻还没下载。你走链接过去时不会有这一下，因为 `<NuxtLink>` 提前替你取了。这一篇讲的就是**不走链接**的那部分跳转该手动补上。

## 官方给的定位

页面标题下面那一句：

> preloadRouteComponents allows you to manually preload individual pages in your Nuxt app.

再往下一段解释它省的是什么：

> Preloading routes loads the components of a given route that the user might navigate to in future. This ensures that the components are available earlier and less likely to block the navigation, improving performance.

关键词是 **the components of a given route**——按路由取组件，而不是按组件名取。

紧跟着有一条 `::tip`，这句决定了这个函数什么时候才轮得到你用：

> Nuxt already automatically preloads the necessary routes if you're using the `NuxtLink` component.

也就是说，链接导航这条路上框架已经做完了。**这个函数服务的是"编程式跳转"**：`navigateTo`、提交后重定向、条件跳板这类没有 `<a>` 元素可以被观察的场景。

## 示例里那行注释才是重点

官方给的用法示例是这样的（注释是原文）：

```ts
// we don't await this async function, to avoid blocking rendering
// this component's setup function
preloadRouteComponents('/dashboard')

const submit = async () => {
  const results = await $fetch('/api/authentication')

  if (results.token) {
    await navigateTo('/dashboard')
  }
}
```

这段里有三个可以照着抄的决策：

**调用点提前到"还不确定要不要跳"的时候**。这里放在 setup 阶段而不是提交成功之后，趁用户在填表、在等响应，把目标页的代码先取回来。

**不 await**。官方注释写明了理由：await 会**阻塞当前组件 setup 的渲染**。预热是"顺手做"的事，做不完也不能拖累当前这一屏。对照 444 那两个组件级函数——它们的示例是带 `await` 的——这是两个入口在文档层面唯一明显的用法差异。

**参数直接写路径字符串**，和 `navigateTo` 的参数形状一致。

## 三条边界

| 事项 | 官方依据 |
| --- | --- |
| 服务端无效 | `::note` "On server, `preloadRouteComponents` will have no effect." |
| 用 `<NuxtLink>` 时不需要它 | `::tip` "Nuxt already automatically preloads the necessary routes..." |
| 不能挡住当前渲染 | 示例注释"to avoid blocking rendering this component's setup function" |

和 444 那条对照一下，两者分工很清楚：

| 入口 | 预热对象 | 命名要求 | 示例是否 await |
| --- | --- | --- | --- |
| `preloadComponents` / `prefetchComponents` | 单个全局注册的组件 | 必须 PascalCase 组件名 | 都带 `await` |
| `preloadRouteComponents` | 一条路由的组件 | 传路由路径 | **故意不 await** |

## 文档没写的部分

这一页没有参数表，也没有签名段，所以这几件事**文档未说明**：

- 是否接受 Vue Router 那种路由位置对象（`{ name: 'dashboard' }`），页面里只出现过路径字符串。
- 预热的范围除了页面组件，是否包含这条路由的**布局和中间件**。442 讲 `prefetch` 属性时官方明写过"will prefetch middleware, layouts and payloads"，但那说的是 `<NuxtLink>` 的预取链路，不能直接算到这个函数头上。
- 传一个匹配不到任何路由的路径会怎样。

要确认第二件事，做法很直接：在一个 `/dashboard` 页面有独立布局的项目里调用它，看网络面板除了页面 chunk 之外还取了哪些文件；再对照用 `<NuxtLink to="/dashboard">` 时取到的清单。

## 自查

- 按路由预热，参数用路径字符串；调用点放在"用户还没决定跳不跳"的时候。
- **不要 await 它**，官方注释就是这条理由。
- 用了 `<NuxtLink>` 的跳转不需要手动预热，这个函数只补编程式跳转。
- 服务端调用无效果，别放在服务端插件里。
- 是否含布局与中间件、能否传路由名对象，文档未说明，用网络面板对照 `<NuxtLink>` 实测。

## 下一篇

下一篇 446 讲加载态的一个反直觉默认值：`useAsyncData` 和 `useFetch` 返回的 `pending`，在默认配置下 `status` 为 `idle` 时是 `false` 的——`{ server: false }` 在服务端渲染时、`{ immediate: false }` 都属这种"看着在等、其实没在飞"的时刻；官方给的话是"要区分 idle 和在飞，就用 `status`"。

## 本文依据

- `preloadRouteComponents` 全页：标题下的人工预热定位、"loads the components of a given route ... less likely to block the navigation" 段、`::tip` 关于用 `NuxtLink` 时框架已自动预取、`navigateTo` 示例及其不 await 的注释原文、`::note` 服务端无效、指向 `navigateTo` 的 read-more、源文件指向 `composables/preload.ts`（`api/utils/preload-route-components`）
- `prefetch` 属性定义中"will prefetch middleware, layouts and payloads"（`api/components/nuxt-link`）
- `preloadComponents`/`prefetchComponents` 的 PascalCase 与 await 用法（`api/utils/preload-components`、`api/utils/prefetch-components`）
