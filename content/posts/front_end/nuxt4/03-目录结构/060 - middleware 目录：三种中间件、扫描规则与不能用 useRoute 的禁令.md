---
url: /posts/d0ab308fd555598abb61894de72b6a33/
title: 060 - middleware 目录：三种中间件、扫描规则与那条不能用 useRoute 的禁令
date: 2026-11-01T09:00:00+08:00
updated: 2026-11-01T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：Nuxt 的路由中间件分三种：匿名（内联）中间件直接写在页面里、命名中间件放在 app/middleware/ 里并在被页面使用时通过异步导入自动加载、全局中间件放在同一目录但带 .global 后缀并在每一次路由变化时运行；前两种可以在 definePageMeta 里定义。目录侧有两条规则：名字规范化为 kebab-case（myMiddleware 变成 my-middleware），且只有目录顶层的文件（或任意子目录里的 index 文件）会被注册——index 文件从包含它的文件夹取名，middleware/auth/index.ts 注册为 auth。执行顺序官方给的是全局中间件先跑、再按页面声明的顺序跑，而全局之间默认按文件名字母序，官方建议用带前导零的字母序编号（01.setup.global.ts 在 02.analytics.global.ts 之前）来控制顺序并提醒文件名是按字符串排序的所以 10.new.global.ts 会排在 2.new.global.ts 之前。返回值契约是这一节最关键的部分：vue-router 的导航守卫会传第三个 next() 参数，而这里不传，重定向或取消导航靠从中间件返回值得到——可以是什么都不返回、navigateTo('/')（服务端时默认 302，可传 redirectCode: 301）、abortNavigation() 或 abortNavigation(error)。还有一条禁令与它的理由：一定要用 to 与 from 参数访问下一个与上一个路由，完全避免在这个上下文里使用 useRoute()，因为中间件里没有"当前路由"这个概念（它可能中止导航或重定向到别的路由），useRoute() 在这个上下文里永远不准确；官方还警告有时你调用的组合式函数内部用了 useRoute()，即使你的中间件里没有直接调用也会触发这个警告。

categories:
  - 前端开发

tags:
  - nuxt4
  - 中间件
  - 路由守卫
  - 执行顺序
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

中间件是最容易写错的一层，原因不在逻辑难度，而在于它的接口和 `vue-router` 长得不一样、却让人以为一样。这一节官方把四处差别都写明了，三种类型、扫描规则、返回值契约、以及那条 `useRoute()` 禁令。

三种中间件，和一条它不是服务端中间件。官方的分类原文是，路由中间件有三种。

一，匿名或内联路由中间件，直接定义在页面内部。

二，命名路由中间件，放在 `app/middleware/`，在被页面使用时通过异步导入自动加载。

三，全局路由中间件，放在 `app/middleware/` 并带一个 `.global` 后缀，在每一次路由变化时运行。

前两种路由中间件可以在 `definePageMeta` 里定义。

然后那条 note 澄清了一件同名混淆，路由中间件运行在你 Nuxt 应用的 Vue 那一侧。尽管名字相似，它们与服务端中间件完全不同，后者运行在你应用的 Nitro 服务端部分。

这张对照表建议直接背下来。`app/middleware/*.global.ts` 跑在 Vue 侧，触发时机是每一次路由变化。`app/middleware/auth.ts`，命名，跑在 Vue 侧，页面在 `definePageMeta` 里点名它时触发。`server/middleware/*.ts` 跑在 Nitro 服务端，触发时机是服务端请求处理链上。

050 那张目录表里两处都出现过 `middleware`，就是这一条。027 那个 `render:html` 插件是服务端那一侧的，别和这里混。

两条命名与扫描规则。

规范化，官方原话，中间件的名字会被规范化为 kebab-case，`myMiddleware` 变成 `my-middleware`。

026、059 讲布局时是同一条规则，框架里文件名与名字规范化为 kebab-case 是通用的。

扫描范围，这条最容易被踩。官方原话，只有位于该目录顶层的文件，或任意子目录中的 index 文件，会被注册。一个 index 文件从包含它的那个文件夹取得自己的名字，`middleware/auth/index.ts` 被注册为 `auth`。

读出的操作规则有两条。

子目录里的普通文件不会生效。`middleware/auth/session.ts` 不会被注册成 `auth-session` 之类的名字，只有 `index` 那个文件算。

想按目录组织，就用 `目录/index.ts`，它注册名是目录名。

用法，签名是 `(to, from)`，返回值才是控制手段。官方原话，路由中间件是接收当前路由与下一个路由作为参数的导航守卫。

官方给的完整示例，注意注释里那条警告。

```ts [middleware/my-middleware.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  if (to.params.id === '1') {
    return abortNavigation()
  }
  // In a real app you would probably not redirect every route to `/`
  // however it is important to check `to.path` before redirecting or you
  // might get an infinite redirect loop
  if (to.path !== '/') {
    return navigateTo('/')
  }
})
```

示例注释里那句必须先检查 `to.path` 否则会拿到无限重定向循环，是这一节最值钱的一行实操提醒。

然后是那句与 `vue-router` 的关键差别，与 `vue-router` 的导航守卫不同，不会传入第三个 `next()` 参数，而重定向或取消导航是靠从中间件返回一个值来处理的。

官方给出的返回值全集，做成表是这样。

什么都不返回，简单 `return` 或不 return，效果是不阻塞导航，进入下一个中间件函数，如果有，或完成这次路由导航。

`return navigateTo('/')`，重定向到给定路径，如果发生在服务端，会把重定向码设为 `302`。

`return navigateTo('/', { redirectCode: 301 })`，重定向并把服务端重定向码设为 `301` Moved Permanently。

`return abortNavigation()`，停止当前导航。

`return abortNavigation(error)`，以错误拒绝当前导航。

这一张表里两个新手必踩的点。

忘了 return 不等于阻止导航，而是放行。写了 `navigateTo('/')` 却没 `return`，跳转不会发生。

重定向码只在服务端那一次有意义，官方两处都写着 if the redirect happens on the server side。

官方还留了一条 important 收边，我们建议使用上面的助手函数来执行重定向或停止导航。`vue-router` 文档里描述的其它可能返回值也许能用，但未来可能发生破坏性变更。

执行顺序，全局先跑，全局之间按文件名字母序。官方原话，中间件按以下顺序运行，一，全局中间件，二，页面定义的中间件顺序，如果用数组语法声明了多个中间件。

官方给的例子很完整。

```bash [app/middleware/ directory]
-| middleware/
---| analytics.global.ts
---| setup.global.ts
---| auth.ts
```

```vue [pages/profile.vue]
<script setup lang="ts">
definePageMeta({
  middleware: [
    function (to, from) {
      // Custom inline middleware
    },
    'auth',
  ],
})
</script>
```

官方的预期顺序是，`analytics.global.ts`，`setup.global.ts`，自定义内联中间件，`auth.ts`。

注意第三、四项的顺序来自数组里的书写顺序，内联函数写在 `'auth'` 之前，所以先跑。

全局之间怎么控制，官方给了办法和它的坑。默认情况下，全局中间件基于文件名按字母序执行。不过有时你想定义一个特定顺序。例如上面这个场景里，`setup.global.ts` 可能需要先于 `analytics.global.ts` 运行。那种情况下，我们建议给全局中间件加上字母序的数字前缀。

```bash [Directory structure]
-| middleware/
---| 01.setup.global.ts
---| 02.analytics.global.ts
---| auth.ts
```

官方跟着一条 note，如果你是第一次接触字母序编号，请记住文件名是作为字符串排序的，不是作为数值。例如 `10.new.global.ts` 会排在 `2.new.global.ts` 之前。这就是为什么上面的例子给个位数编号加了 `0`。

这一条我建议你直接抄那个前导零，`01.`、`02.`。超过九个全局中间件时还要继续补零，`09.` 之后是 `10.`，再往后按字符串排会乱，这是我对官方这条规则的推演，官方只举了个位数加 `0` 的例子。

中间件什么时候跑，服务端渲染过一遍、客户端再来一遍。官方原话，如果你的站点是服务端渲染或生成的，初始页面的中间件会在页面被渲染时执行一次，然后在客户端再执行一次。这可能正是你的中间件所需要的，例如你的中间件需要一个浏览器环境，比如你是生成的站点、积极缓存响应，或者你想从 local storage 读一个值。

不过如果你想避免这个行为，官方给了三个精确的跳过档位。

```ts [middleware/example.ts]
export default defineNuxtRouteMiddleware((to) => {
  // skip middleware on server
  if (import.meta.server) {
    return
  }

  // skip middleware on client side entirely
  if (import.meta.client) {
    return
  }

  // or only skip middleware on initial client load
  const nuxtApp = useNuxtApp()
  if (import.meta.client && nuxtApp.isHydrating && nuxtApp.payload.serverRendered) {
    return
  }
})
```

这三个档位是，只在服务端跳过、在客户端整个跳过、以及只在首次客户端加载时跳过，也就是水合中且服务端已渲染。第三个条件用得最讲究，它靠 `nuxtApp.isHydrating` 与 `nuxtApp.payload.serverRendered` 两个标志组合。

然后是一句会让人意外的补充，即使你在服务端于中间件里抛出一个错误、并渲染了错误页，这一点依然成立，中间件仍会在浏览器里再运行一次。

官方接着用 note 解释了原因并给了对策，渲染错误页是一次完全独立的页面加载，这意味着任何已注册的中间件都会再次运行。你可以在中间件里使用 `useError` 来检查是否正在处理一个错误。

这就是我在中间件里做的重定向，怎么在 404 页上又跑了一遍的官方答案。056 那个 `error.vue` 不属于路由，它没有 `definePageMeta`，但中间件会照跑，判据要用 `useError`。

那条禁令，不要用 `useRoute()`。官方这一节的措辞是整页最重的，始终使用你中间件里的 `to` 与 `from` 参数来访问下一个与上一个路由。在这个上下文里完全避免使用 `useRoute()` 组合式函数。中间件里没有当前路由这个概念，因为中间件可能中止一次导航、或重定向到另一个路由。`useRoute()` 组合式函数在这个上下文里永远不会准确。

注意永远不会准确这几个字。它不是不建议，而是概念上不成立。

而且官方预见了间接命中的情况，有时候，你会调用一个内部使用了 `useRoute()` 的组合式函数，即使你的中间件里没有直接调用它，也会触发这个警告。这会导致与上面完全相同的问题，所以你应该把函数组织成接收路由作为参数的形式，以便在中间件里使用。

官方给的三个文件正好是正确、正确支持、错误这一组。

```ts [middleware/access-route.ts]
export default defineNuxtRouteMiddleware((to) => {
  // passing the route to the function to avoid calling `useRoute()` in middleware
  doSomethingWithRoute(to)

  // ❌ this will output a warning and is NOT recommended
  callsRouteInternally()
})
```

```ts [utils/handle-route.ts]
// providing the route as an argument so that it can be used in middleware correctly
export function doSomethingWithRoute (route = useRoute()) {
  // ...
}
```

```ts [utils/dont-do-this.ts]
// ❌ this function is not suitable for use in middleware
export function callsRouteInternally () {
  const route = useRoute()
  // ...
}
```

第二个文件这个写法值得学，`route = useRoute()` 用默认参数。普通页面里调用它可以不传参，自己拿 `useRoute()`，中间件里调用它必须把 `to` 传进去。一个函数两侧都能正确工作，靠的是把路由变成参数而不是内部去取。

在插件里动态加中间件。官方原话，可以手动添加全局或命名路由中间件，使用 `addRouteMiddleware()` 助手函数，例如在插件内部。

```ts
export default defineNuxtPlugin(() => {
  addRouteMiddleware('global-test', () => {
    console.log('this global middleware was added in a plugin and will be run on every route change')
  }, { global: true })

  addRouteMiddleware('named-test', () => {
    console.log('this named middleware was added in a plugin and would override any existing middleware of the same name')
  })
})
```

注意官方在第二个回调里写的那句 self-description，它会覆盖任何同名的已存在中间件。这条覆盖行为是这句注释给出的，插件里加的同名中间件会压过 `app/middleware/` 里那个。

060 与 061 的交界就在这儿，`addRouteMiddleware` 在插件里调用，插件本身的注册顺序与依赖在 061 讲。

在构建期给页面挂中间件。官方原话，与其在每个页面里使用 `definePageMeta`，你可以在 `pages:extend` 钩子里添加命名路由中间件。

```ts [nuxt.config.ts]
import type { NuxtPage } from 'nuxt/schema'

export default defineNuxtConfig({
  hooks: {
    'pages:extend' (pages) {
      function setMiddleware (pages: NuxtPage[]) {
        for (const page of pages) {
          if (/* some condition */ Math.random() > 0.5) {
            page.meta ||= {}
            // Note that this will override any middleware set in `definePageMeta` in the page
            page.meta.middleware = ['named']
          }
          if (page.children) {
            setMiddleware(page.children)
          }
        }
      }
      setMiddleware(pages)
    },
  },
})
```

这里有一条 important 级别的行为说明藏在注释里，注意，这会覆盖页面里 `definePageMeta` 设置的任何中间件。

两条通道同名时的优先级，官方用这一行注释给了答案，`pages:extend` 赢。017 讲约定的代价时提过这类配置层覆盖文件层的现象。

## 自查

- [ ] `middleware/auth/session.ts` 会被注册吗？改成什么文件名才会？
- [ ] 中间件里不写 `return`，导航是被阻止还是被放行？
- [ ] `navigateTo('/login')` 在服务端产生的状态码默认是什么？怎么改成 301？
- [ ] 两个全局中间件谁先跑？为什么官方给的编号是 `01.` 而不是 `1.`？
- [ ] 服务端已经跑过一遍中间件了，为什么客户端还要跑一遍？怎么只跳过首次水合那次？
- [ ] 你的中间件调用的组合式函数内部用了 `useRoute()`，会怎样？官方推荐的函数签名怎么改？
- [ ] `pages:extend` 里设的中间件与页面里 `definePageMeta` 设的，谁生效？

## 下一篇

061 讲 `app/plugins/`：官方给的注册顺序规则（含子目录 index 那条已废弃的写法）、`enforce` 与 `order` 两种排序手段，以及插件里哪些组合式函数不能用。
