---
url: /posts/423ccc41a05d53dd02945bf7909e000b/
title: 065 - server 地图：api、routes、middleware、plugins、utils、types
date: 2026-11-02T11:00:00+08:00
updated: 2026-11-02T11:00:00+08:00
author: cmdragon

summary:
  摘要：server/ 目录用来给你的应用注册 API 与服务端处理器，Nuxt 会自动扫描这些目录里的文件并以 HMR 支持注册，官方给的树把三件事一次说清：server/api/hello.ts 对应 /api/hello、server/routes/bonjour.ts 对应 /bonjour、server/middleware/log.ts 记录所有请求。每个文件应导出一个用 defineEventHandler() 或它的别名 eventHandler() 定义的默认函数，处理器可以直接返回 JSON 数据、一个 Promise，或用 event.node.res.end() 发送响应。这一节有一条 important 级别的边界：不要在你的服务端路由或工具函数里导入 Vue 应用侧代码（组件、组合式函数或其它 app-only 工具），也不要在应用里导入服务端专用代码。另外三条规则各管一个子目录：api 里的文件路径会自动加 /api 前缀而 routes 不加，服务端中间件会在任何其它服务端路由之前对每个请求运行但官方要求它不应返回任何东西（也不关闭或响应请求）、只能检查或扩展请求上下文或抛出错误，server/plugins 里的文件被注册为 Nitro 插件以便扩展运行时行为并挂钩生命周期事件。v4.3 起还有 #server 别名可以从 server/ 内任意位置导入，但官方明确它只能在 server/ 目录内使用，在客户端代码里从 #server 导入会导致一个错误。

categories:
  - 前端开发

tags:
  - nuxt4
  - 服务端
  - nitro
  - h3
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

063 末尾那条只有 `server/utils` 会在 `server/` 里被自动导入，就是这个目录的边界感。这一篇把 `server/` 摊开，六个子目录各自的规则、处理器契约，以及那条跨侧导入的禁令。

扫描与契约，先建立形状感。官方原话，`server/` 目录用于给你的应用注册 API 与服务端处理器。Nuxt 会自动扫描这些目录里的文件，以带热模块替换，HMR，支持的方式注册 API 与服务端处理器。

官方给的树把路径到 URL 三件事一起说了。

```bash [Directory Structure]
-| server/
---| api/
-----| hello.ts      # /api/hello
---| routes/
-----| bonjour.ts    # /bonjour
---| middleware/
-----| log.ts        # log all requests
```

然后是处理器契约，两句。每个文件应当导出一个用 `defineEventHandler()` 或 `eventHandler()`，别名，定义的默认函数。处理器可以直接返回 JSON 数据、一个 `Promise`，或者使用 `event.node.res.end()` 发送响应。

```ts [server/api/hello.ts]
export default defineEventHandler((event) => {
  return {
    hello: 'world',
  },
})
```

官方说，现在你可以在你的页面与组件里通用地，universally，调用这个 API。

```vue [app/pages/index.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/hello')
</script>

<template>
  <pre>{{ data }}</pre>
</template>
```

这一对文件正好把 028 那条闭环补上了，028 我借用了 `/api/count` 这种路径讲取数，接口本身从这一篇开始有了出处，`server/api/hello.ts` 返回一个对象，`useFetch('/api/hello')` 就能拿到。

那条 important，两侧代码不许互引。官方用 important 给了一条硬边界，不要在你的服务端路由或工具函数里导入 Vue 应用侧代码，组件、组合式函数或其它 app-only 工具，也不要在你的应用里导入服务端专用代码。

官方紧接着给了为什么的去处，read-more 指向 shared 页的 Why You Cannot Mix Vue and Nitro Code 一节，066 讲。

这一条与 063 那条 important 是同一件事的两面。app/utils 页那句是，这些 utils 只在你的应用的 Vue 那一部分可用，只有 `server/utils` 会在 `server/` 目录里被自动导入。server 页这句是，不要导入 Vue 应用侧代码进服务端路由与工具，不要导入服务端专用代码进应用。

要两侧共用，位置是 `shared/`，066 讲。

`api/` 与 `routes/`，唯一的差别是那个前缀。官方原话，位于 `~~/server/api` 里的文件，它们的路径会自动加上 `/api` 前缀。要添加不带 `/api` 前缀的服务端路由，把它们放进 `~~/server/routes` 目录。

```ts [server/routes/hello.ts]
export default defineEventHandler(() => 'Hello World!')
```

官方说，以上例为准，`/hello` 路由可以在 http://localhost:3000/hello 访问到。

判据就这一句，需要 `/api` 命名空间，放 `api/`，要一个自定义根路径，官方举的例子是动态的 `/sitemap.xml`，050 那张表里写的也是它，放 `routes/`。

但这一节末尾有一条限制必须知道，请注意，目前服务端路由并不支持像页面那样完整的动态路由功能。

这句是我照 058 那套方括号语法写服务端路由、结果不对的官方解释，页面的那套动态段规则不整套适用于这里。支持到什么程度，这一节没给清单。

`middleware/`，跑在所有请求之前，但不许返回东西。官方原话，Nuxt 会自动读取 `~~/server/middleware` 里的任何文件，为你的项目创建服务端中间件。中间件处理器会在任何其它服务端路由之前、对每一个请求运行，用来添加或检查请求头、记录请求，或扩展事件 request 对象。

然后是用 note 写的禁令，中间件处理器不应返回任何东西，也不关闭或响应请求，它们只应当检查或扩展请求上下文，或者抛出一个错误。

官方给的两个例子正好对应记录和扩展上下文。

```ts [server/middleware/log.ts]
export default defineEventHandler((event) => {
  console.log('New request: ' + getRequestURL(event))
})
```

```ts [server/middleware/auth.ts]
export default defineEventHandler((event) => {
  event.context.auth = { user: 123 }
})
```

第二个例子是这条链的关键，往 `event.context` 上挂东西，后面的处理器再读它。这就是服务端中间件怎么把信息交给路由的官方写法。

注意它与 060 讲的路由中间件是两套完全不同的东西，那一节 note 已经明说尽管名字相似，它们完全不同。这里在每个请求上跑、不许返回值，那里在导航到某个路由之前跑、靠返回值控制重定向与取消。

`plugins/`，这里注册的是 Nitro 插件。官方原话，Nuxt 会自动读取 `~~/server/plugins` 目录里的任何文件，并把它们注册为 Nitro 插件。这让你能够扩展 Nitro 的运行时行为并挂钩到生命周期事件。

```ts [server/plugins/nitroPlugin.ts]
export default defineNitroPlugin((nitroApp) => {
  console.log('Nitro plugin', nitroApp)
})
```

027 那个改 HTML 的 `render:html` 插件就是这一节的用法，官方当时给的文件路径也是 `server/plugins/extend-html.ts`。对照记一次，`app/plugins/` 注册成 Nuxt 插件、用 `defineNuxtPlugin`、时机是 Vue 应用创建时，`server/plugins/` 注册成 Nitro 插件、用 `defineNitroPlugin`、时机是 Nuxt 服务器创建时，050 的原话。

061 那整页规则，顺序编号、`parallel`、对象式静态分析，不适用于这里，那是应用侧插件的机制。

`utils/`，h3 已经带了一批助手，这里加你自己的。官方原话，服务端路由由 h3 驱动，它带有一组顺手的助手。你可以在 `~~/server/utils` 目录里自己添加更多助手。例如，你可以定义一个自定义处理函数工具，它包住原来的处理器、在返回最终响应之前执行额外操作。

官方给的这个例子值得读完，因为它是服务端也要横切关注点的写法。

```ts [server/utils/handler.ts]
export const defineWrappedResponseHandler = <T extends EventHandlerRequest, D> (
  handler: EventHandler<T, D>,
): EventHandler<T, D> =>
  defineEventHandler<T>(async (event) => {
    try {
      // do something before the route handler
      const response = await handler(event)
      // do something after the route handler
      return { response }
    } catch (err) {
      // Error handling
      return { err }
    }
  })
```

```ts [server/api/hello.get.ts]
export default defineWrappedResponseHandler(event => 'hello world')
```

注意第二个文件的文件名，`hello.get.ts`，这就带出了后面方法后缀那一节。

`types/`，只服务端可见，而且只扫这一层。官方原话，放在 `~~/server/types/` 里的类型只在服务端上下文里被自动导入，所以你可以在服务端路由、中间件、插件与工具函数里引用它们而无需 import。同样需要在 Vue 应用里用的类型，则应当属于 `shared/types/`。

```ts [server/types/todo.ts]
export interface Todo {
  id: string
  title: string
  completed: boolean
}
```

```ts [server/api/todos.get.ts]
export default defineEventHandler((): Todo[] => {
  return []
})
```

官方接着说，只有直接位于 `server/types/` 里的文件会被扫描，嵌套子目录里的文件不会被自动导入，这与 `shared/types/` 的工作方式一致。

这句与 `shared/types/` 一致很重要，063 讲的 `composables/nested/utils.ts` 不扫描、这里是同一族规则，而 `server/plugins/`、`server/api/` 这些是递归生成路由的。扫顶层与递归建路由是两套不同的事，别混。

`#server` 别名，v4.3。官方原话，你可以使用 `#server` 别名从 `server/` 目录内的任何位置导入文件，无论导入方文件在哪里。

```ts [server/api/users/[id]/profile.ts]
// Instead of relative paths like this:
// import { formatUser } from '../../../utils/formatUser'

// Use the #server alias:
import { formatUser } from '#server/utils/formatUser'
```

官方说，这个别名确保了你服务端代码里一致的导入，在深层嵌套的路由处理器里尤其有用。

官方跟着一条 note，`#server` 别名只能在 `server/` 目录内使用。在客户端代码里从 `#server` 导入会导致一个错误。

这三段连读很顺，051 那张默认别名表里 `#server` 就指向 `<rootDir>/server`，而这一节的 note 说的是客户端里用它会报错，这正是 065 开头那条 important 禁令在导入路径层面的表现。

文件名后缀，`.get` 与 `.post` 这些是真的生效。官方原话，处理文件的名称可以用 `.get`、`.post`、`.put`、`.delete` 这些作为后缀来匹配请求的 HTTP 方法。

```ts [server/api/test.get.ts]
export default defineEventHandler(() => 'Test get handler')
```

```ts [server/api/test.post.ts]
export default defineEventHandler(() => 'Test post handler')
```

官方说，以上例为准，用 GET 方法取 `/test`，返回 `Test get handler`，用 POST，返回 `Test post handler`，任何其它方法，返回 405 错误。

官方接着说，你也可以在目录里使用 `index.[method].ts` 来换一种组织方式，这对创建 API 命名空间很有用。

官方给的三个文件把三种形态并列了，`server/api/foo/index.get.ts` 是 `api/foo` 的 GET，`server/api/foo/index.post.ts` 是 POST，`server/api/foo/bar.get.ts` 是 `api/foo/bar` 的 GET。

方法与 body 的关系，官方在 Body Handling 那条 note 里连上了，我们在文件名里用 `submit.post.ts`，只是为了匹配能够接收请求体的 `POST` 请求。当在 GET 请求里使用 `readBody` 时，`readBody` 会抛出一个 `405 Method Not Allowed` HTTP 错误。

动态段在 `api/` 里怎么写。官方原话，服务端路由可以在文件名里使用方括号内的动态参数，例如 `/api/hello/[name].ts`，并通过 `event.context.params` 访问。

```ts [server/api/hello/[name].ts]
export default defineEventHandler((event) => {
  const name = getRouterParam(event, 'name')

  return `Hello, ${name}!`
})
```

官方说，现在你可以通用地调用 `/api/hello/nuxt` 这个 API 并得到 `Hello, nuxt!`。

官方还给了 tip，改用 `getValidatedRouterParams` 配一个像 Zod 或 Valibot 这样的 schema 校验器，可以同时拿到运行时与类型安全，query 与 body 也各有一条同样的 tip，`getValidatedQuery`、`readValidatedBody`。

catch-all 在这里的写法与页面略有不同，两种形态官方都给了。

```ts [server/api/foo/[...].ts]
export default defineEventHandler((event) => {
  // event.context.path to get the route path: '/api/foo/bar/baz'
  // event.context.params._ to get the route segment: 'bar/baz'
  return `Default foo handler`
})
```

官方说，你可以用 `~~/server/api/foo/[...slug].ts` 给这个 catch-all 路由起一个名字，并通过 `event.context.params.slug` 访问它。

`[...]` 与 `[...slug]` 的差别就在参数名，前者落在 `event.context.params._` 上，官方注释点明了。

## 自查

- [ ] `server/routes/hello.ts` 与 `server/api/hello.ts` 分别对应哪个 URL？
- [ ] 服务端中间件为什么不能返回值？它靠什么把信息交给后面的处理器？
- [ ] `server/plugins/` 与 `app/plugins/` 各注册成什么、用哪个宏？
- [ ] `server/types/nested/x.ts` 会被自动导入吗？想在 Vue 应用里也用到某类型该放哪？
- [ ] 在页面组件里 `import ... from '#server/...'` 会发生什么？
- [ ] `test.get.ts` 加 `test.post.ts` 之外用 DELETE 请求会拿到什么状态码？
- [ ] 服务端路由的动态段比页面那套弱在哪？官方原话怎么说？

## 下一篇

066 讲 `shared/`：官方给的那个「为什么不能混用 Vue 与 Nitro 代码」的解释就在这里，而它同时也是 065 那条 important 禁令的原因。
