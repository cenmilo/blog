---
url: /posts/08110829b8633253a1d74e9dad66e9be/
title: 529 - CDN 代理要关掉的两项：Rocket Loader 与邮箱地址混淆
date: 2027-04-06T10:00:00+08:00
updated: 2027-04-06T10:00:00+08:00
author: cmdragon

summary:
  摘要：部署页的 CDN Proxy 一节把 Cloudflare 的"压缩与安全选项"点名为风险来源，要求关掉两项：**Speed > Settings > Content Optimization > Rocket Loader™** 与 **Security > Settings > Email Address Obfuscation**。给出的理由是这些功能会往你的应用里注入脚本，而后果官方写得很具体：**unnecessary re-rendering or hydration errors**（多余的重渲与水合错误）。同节还有一句提醒：这两项在 Cloudflare 控制台里的位置有时会变，找不到就多翻翻。这一条与"用 CDN 分发静态资源"是两件不同的事——后者靠 `app.cdnURL`，只改资源地址、不动 HTML 内容。

categories:
  - 前端开发

tags:
  - nuxt4
  - cdn
  - cloudflare
  - 水合
  - 部署
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上线后有些用户报"页面点了没反应"，你自己访问一切正常。开过 Cloudflare 的站点里，这一类问题的原因可能根本不在你的代码：CDN 在把 HTML 发给浏览器之前改了它。

官方在部署页专门为这件事写了一节。

## 那一节说了什么

标题是 CDN Proxy，第一段给的是适用范围与风险来源：

> In most cases, Nuxt can work with third-party content that is not generated or created by Nuxt itself. But sometimes such content can cause problems, especially Cloudflare's "Minification and Security Options".

第二段是要求与后果：

> Accordingly, you should make sure that the following options are unchecked / disabled in Cloudflare. Otherwise, unnecessary re-rendering or hydration errors could impact your production application.

要关的两项，官方给的菜单路径逐字如下：

1. Speed > Settings > Content Optimization > Disable "Rocket Loader™"
2. Security > Settings > Disable "Email Address Obfuscation"

同节末尾两句，一句解释一句提醒：

> With these settings, you can be sure that Cloudflare won't inject scripts into your Nuxt application that may cause unwanted side effects.

**这两项被关掉的真实动机是"别让 CDN 往你的 HTML 里注入脚本"**，而那条 `::tip` 是：它们的控制台位置有时会变，所以要**多翻找一下**（原文：Their location on the Cloudflare dashboard sometimes changes so don't hesitate to look around）。

## 为什么注入脚本会打到 Nuxt

先把水合说清楚：服务端先把 HTML 拼好发给浏览器，浏览器接手这段 HTML、把事件与状态挂上去，这一步就是水合。**它的前提是"浏览器看到的 HTML 与 Vue 期望的那份结构对得上"。**

官方在这一节里给的两个后果正落在这条链上：

- **水合错误**：注入进来的脚本改了 DOM，DOM 就不再是渲染输出的那个形状，挂接过程出错。
- **多余的重渲**：内容被改动后，应用要重做工作——官方没有细说重渲的具体表现，只把它列为会影响生产应用。

至于 Rocket Loader 与邮箱地址混淆分别是怎么改的，官方没写：前者按 Cloudflare 的功能名归在"Content Optimization"下，后者归在 Security 下；这一节里**唯一的统一说明是那句"会注入脚本"**。

## 与 cdnURL 是两件事

这里最容易混，因为都叫 CDN。两条路完全不同：

| 你要做的事 | 手段 | 会不会动 HTML |
| --- | --- | --- |
| 让 `public/` 里的静态资源从 CDN 域名取 | `app.cdnURL`（或启动时用 `NUXT_APP_CDN_URL`） | **不会**，只是资源 URL 的拼接规则变了 |
| CDN 在响应里注入脚本、压缩改写 | 平台侧功能（如 Rocket Loader、邮箱地址混淆） | **会**，这正是本节要求关掉的原因 |

换句话说：把资源交给 CDN 服务是配置问题，安全；让 CDN 加工你的 HTML 才是本节的风险来源。用 `cdnURL` 之前那篇已经讲过它的解析链（`cdnURL` 优先、回退 `app.baseURL`，`app.head` 里写死的路径不参与）。

## 怎么确认你的站点被改了

官方没有给检测步骤。可行的取证办法是**比较同一份 HTML 经过 CDN 前后有没有多出内容**：

- 直接请求源站拿一份响应，再请求 CDN 域名拿一份，比较 `<script>` 标签数量与位置。
- 浏览器里对比"查看网页源代码"里的 HTML 与开发者工具 Elements 面板里最初的 DOM 结构，注意有没有不属于你构建产物的脚本节点。

**上面两条是本系列给的排查思路，官方在文档里只给了要关哪两项，没给检测方法。** 结论落点很简单：如果确认是 CDN 注入造成的，处理方式是关掉对应功能，而不是去改你的应用代码——因为改不动，HTML 在到达浏览器之前就被替换了。

## 自查

- 官方点名的风险来源是 Cloudflare 的压缩与安全选项，动作是关掉两项：Rocket Loader（Speed > Settings > Content Optimization）与邮箱地址混淆（Security > Settings）。
- 后果只有两句：**多余重渲与水合错误**；机制是"注入脚本"，具体怎么注入的两项各自说明官方没写。
- 控制台位置会变，找不到就翻——这是官方自带的提醒。
- 与 `app.cdnURL` 分开看：那是资源 URL 的事，不动 HTML；本节说的是 CDN 改写响应内容。
- 检测办法文档没给，可比对源站与 CDN 两份响应；确认被注入后要改的是平台配置，不是应用代码。

## 下一篇

510-529 到这里收尾。下一篇 530 进 530-599 那几篇，开始把 `nuxt.config.ts` 的选项参考逐节读完：这一节有多少顶层节、三层结构长什么样、以及它连开场段都没有这个事实。
