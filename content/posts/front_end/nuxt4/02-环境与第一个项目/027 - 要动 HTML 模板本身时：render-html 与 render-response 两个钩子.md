---
url: /posts/e81924574f330b899013b4b366bcd950/
title: 027 - 要动 HTML 模板本身时：render-html 与 render-response 两个钩子
date: 2026-10-21T09:00:00+08:00
updated: 2026-10-21T09:00:00+08:00
author: cmdragon

summary:
  摘要：当 <head> 层面的声明式配置不够用、你必须直接改发给浏览器的 HTML 时，官方给的路径不是改某个 .html 文件，而是"加一个注册了钩子的 Nitro 插件"。Views 页的 Advanced 小节给的正是这一条：在 server/plugins/extend-html.ts 里用 defineNitroPlugin，注册 render:html 钩子，在 HTML 发给客户端之前修改它——文档的例子是 html.head.push('<meta name="description" ... />')，Hooks 页的例子是 html.bodyAppend.push('<hr>Appended by custom plugin')；同一份插件里还可以注册 render:response 来拦截最终响应。官方在开头就提醒：只需要改 <head> 的话，去看 SEO 与 meta 那一节，别用这条路。

categories:
  - 前端开发

tags:
  - nuxt4
  - 钩子
  - nitro
  - html模板
  - 服务端插件

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前两篇我们在 `app.vue`、`components/`、`pages/`、`layouts/` 这几层里活动。这一篇讲的是**这几层之外**的一件事：渲染完成之后、HTML 发给浏览器之前，最后那一道口子。

先说清楚：**这条路你大概率用不上。** 官方在小节开头第一件事就是劝退。

## 官方的第一句是"先别这么做"

Views 页 Advanced: Extending the HTML Template 小节开头：

> 如果你只需要修改 `<head>`，请参考 SEO and meta 那一节。

这句是分流用的。**你要加 meta、title、link 标签——那是声明式配置（`useHead` / `<Head>` 那一套）的活，不是这里的活。** 只有当你需要的东西不在"往 head 里放标签"这个范围内，比如要在 `<body>` 末尾追加内容、要在 HTML 离开服务器前整体看一眼，才继续往下读。

## 官方的做法：加一个 Nitro 插件，注册钩子

> 你可以通过**添加一个注册了钩子的 Nitro 插件**，获得对 HTML 模板的完全控制。
>
> `render:html` 钩子的回调函数让你可以在 HTML 发送给客户端之前修改它。

这段话里有两个关键判断：

1. **载体是插件，不是配置文件。** 你写的是一个文件，放在服务器端插件目录里。
2. **时机是"发送给客户端之前"**。也就是说，019 讲的那条服务端链已经跑完、HTML 已经渲染出来了，你现在改的是这份成品。

文档给的完整例子（就是一份文件）：

```ts [server/plugins/extend-html.ts]
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('render:html', (html, { event }) => {
    // This will be an object representation of the html template.
    console.log(html)
    html.head.push(`<meta name="description" content="My custom description" />`)
  })
  // You can also intercept the response here.
  nitroApp.hooks.hook('render:response', (response, { event }) => { console.log(response) })
})
```

## 这两个钩子分别拿到什么

**`render:html`：一个对象形式的 HTML 模板。**

官方在注释里点明了这一点：

> 这将会是 HTML 模板的对象表示（object representation）。

回调签名是 `(html, { event })`：第一个参数是那份对象表示，第二个是一个带 `event` 的对象。例子里的用法是 `html.head.push(...)`——**`head` 是一个数组，往里 push 一个字符串标签**。

Hooks 页给的另一个例子用的是另一个字段：

```ts [~~/server/plugins/test.ts]
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('render:html', (html, { event }) => {
    console.log('render:html', html)
    html.bodyAppend.push('<hr>Appended by custom plugin')
  })

  nitroApp.hooks.hook('render:response', (response, { event }) => {
    console.log('render:response', response)
  })
})
```

`bodyAppend`：往 body 末尾追加。跟 `head.push` 是同一类操作，只是位置不同。这也正是那句"只想改 head 就去看 SEO"的分界——**body 层面的追加，声明式配置管不到，只能走这里。**

**`render:response`：拦截最终响应。**

官方给的注释只有一句："你也可以在这里拦截响应。" 回调签名同样是 `(response, { event })`，例子只 `console.log(response)`。它的用途比 `render:html` 更靠后一层——HTML 已经拼好，现在你要动的是响应本身。

## 这份文件放在哪

注意两个文档给的**路径写法不一样**，但指的是同一类位置：

- Views 页写 `server/plugins/extend-html.ts`
- Hooks 页写 `~~/server/plugins/test.ts`（`~~/` 是路径别名，051 讲）

共同点很关键：**是 `server/plugins/`，前面没有 `app/`**。Hooks 页在标题下那句说明也是这么定位的：

> 这些钩子可供**服务端插件（server plugins）**使用，以介入 Nitro 的运行时行为。

也就是说：**这条能力在服务端那一侧。** 013 讲"同一份代码跑在两个地方"时说过，服务端跑一次、浏览器跑一次；而这个插件只在服务端那一次里生效——它改的正是那份服务端渲染出来的 HTML。**它不会在你的浏览器里执行。**

`defineNitroPlugin` 这个宏是谁提供的、为什么它在 `server/` 下不用 import，属于 065 与自动导入边界那几篇的内容；这里只需知道它是服务端插件的默认导出形态。

## 什么时候真的需要它

把官方给出的两处用法摆在一起，就是全部有据可依的场景：

1. 要往 `<head>` 里塞一个声明式配置放不进去的标签；
2. 要往 `<body>` 末尾追加内容（`bodyAppend`）；
3. 要在响应发出去之前拦一眼整份响应（`render:response`）。

这三条之外，如果你发现自己在用这个钩子做"数据获取"或"状态处理"，那大概率走错门了——那些在 019 那条链上有专门的步骤。

## 自查

- [ ] 你现在的诉求是"加个 meta 标签"还是"改渲染完的 HTML"？前者该去哪一节？
- [ ] `render:html` 的回调第一个参数是什么形态？`head` 和 `bodyAppend` 分别追加到哪里？
- [ ] `render:response` 拿到的又是什么？
- [ ] 这份插件文件应该在 `app/plugins/` 还是 `server/plugins/`？它会在浏览器里执行吗？
- [ ] 你能说出"HTML 发给客户端之前"这个时机，在 019 那条服务端链上的位置吗？

## 下一篇

028 回到主线，把第一屏做成"有数据的页面"：用 `useFetch` 取一次数并显示出来，同时第一次分清 `useFetch` 和 `$fetch` 各自该用在哪。

## 本文依据

- `getting-started/views`（Advanced: Extending the HTML Template：note"只需修改 <head> 时参考 SEO and meta 一节"；"通过添加一个注册钩子的 Nitro 插件获得对 HTML 模板的完全控制"；"render:html 钩子的回调函数允许在 HTML 发送给客户端之前修改它"；示例 server/plugins/extend-html.ts 中 defineNitroPlugin((nitroApp)=>{ nitroApp.hooks.hook('render:html',(html,{event})=>{ 注释"This will be an object representation of the html template."、console.log(html)、html.head.push('<meta name="description" content="My custom description" />') })、注释"You can also intercept the response here."、nitroApp.hooks.hook('render:response',(response,{event})=>{console.log(response)}) })）
- `guide/going-further/hooks`（Server Hooks (Runtime) 一节："这些钩子可供 server plugins 使用以介入 Nitro 的运行时行为"；示例文件路径 ~~/server/plugins/test.ts；render:html 回调中 html.bodyAppend.push('<hr>Appended by custom plugin')；render:response 回调 (response,{event}) 内 console.log(response)）
