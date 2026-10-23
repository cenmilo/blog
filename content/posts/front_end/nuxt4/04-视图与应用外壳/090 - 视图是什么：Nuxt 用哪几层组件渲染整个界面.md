---
url: /posts/742dab4c59020b27a5656ec038f29f30/
title: 090 - 视图是什么：Nuxt 用哪几层组件渲染整个界面
date: 2026-11-11T09:00:00+08:00
updated: 2026-11-11T09:00:00+08:00
author: cmdragon

summary:
  摘要：官方的总纲是一句话——Nuxt 提供若干组件层级来实现应用的用户界面。这一部分就从这句话开始，把"一个 URL 进来、屏幕上出现内容"这件事拆成五层。第一层 `app.vue`：默认情况下 Nuxt 把它当作**入口点**，为应用的**每一个路由**渲染它的内容；官方还回答了 Vue 开发者一定会问的那句"`main.js` 在哪"——Nuxt 在幕后完成这件事；有 `app/pages/` 目录时 `app.vue` 是可选的（Nuxt 会自动包含一个默认的 `app.vue`），而没有 `app/pages/` 目录时 Nuxt **不会**引入 vue-router 依赖，这对做落地页或不需要路由的应用是有用的；一旦有了 pages 目录，就必须用 `<NuxtPage>` 组件来显示当前页面；`app.vue` 里加的任何东西（JS 与 CSS）都是全局的、会被包含进每一个页面。第二层是组件：放在 `app/components/` 里就会在全应用自动可用，无需显式导入。第三层页面：`app/pages/` 里的每个文件代表一个不同路由并展示其内容，启用它需要建一个 `app/pages/index.vue` 并在 `app.vue` 里加 `<NuxtPage />`（或者删掉 `app/app.vue` 使用默认入口）。第四层布局：包裹在页面外层、承载多个页面共用 UI 的 Vue 文件，用 `<slot />` 显示**页面**内容，`app/layouts/default.vue` 会被默认使用，自定义布局通过页面元数据设置，而官方明确建议"如果你的应用只有一个布局，用 `app/app.vue` 加 `<NuxtPage />` 代替"。第五层才是 HTML 模板本身：加一个注册钩子的 Nitro 插件，用 `render:html` 的回调在 HTML 发送给客户端之前改动它，`render:response` 则可以拦截响应。

categories:
  - 前端开发

tags:
  - nuxt4
  - 视图
  - appvue
  - 布局
  - 页面

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


目录讲完了。从这里开始问的是另一件事：**浏览器里那一屏，到底是哪些组件层层拼出来的。** 官方的原话只有一句——Nuxt 提供若干组件层级（several component layers）来实现应用的用户界面。这一篇把这五层一次摆清楚，后面每一篇各展开一层。

## 五层的全貌

| 层 | 位置 | 它负责什么 | 谁来决定显示它 |
| --- | --- | --- | --- |
| HTML 模板 | Nitro 插件里的钩子 | 最终发送给客户端的整份 HTML | `render:html` / `render:response` |
| `app.vue` | `app/app.vue` | 应用的**根组件**、默认**入口点**，为**每一个路由**渲染其内容 | Nuxt 自动 |
| `<NuxtLayout>` | `app/layouts/*.vue` | 包裹页面的外壳，放多页共用的 UI | 页面元数据；`default.vue` 兜底 |
| `<NuxtPage>` | `app/pages/*.vue` | 与具体路由绑定的那部分内容 | 路由匹配结果 |
| 组件 | `app/components/*.vue` | 可复用的界面碎片（按钮、菜单等） | 自动导入，无需显式 import |

这张表从上往下读就是嵌套关系，从下往上读就是"我这段代码该放哪一层"。

## 第一层：`app.vue` 是入口点，而且是全局的

官方定义："app.vue 文件是你的 Nuxt 应用的主组件。"默认情况下，Nuxt 把这个文件当作**入口点**，并为应用的**每一个路由**渲染它的内容。

```vue [app/app.vue]
<template>
  <div>
    <h1>Welcome to the homepage</h1>
  </div>
</template>
```

写过 Vue 的人看这儿一定会问一句，官方也直接答了：**如果你熟悉 Vue，可能会好奇 `main.js`（那个通常用来创建 Vue 应用的）在哪儿——Nuxt 在幕后完成了这件事。**

`app.vue` 有两种形态，差别只在你有没有 `app/pages/` 目录。

**没有 pages 目录**：这时你只需要一个组件，页面就写在这里。

```vue [app/app.vue]
<template>
  <h1>Hello World!</h1>
</template>
```

官方对这一种的说法是：`app/pages/` 目录是可选的，如果它不存在，**Nuxt 不会包含 vue-router 依赖**——这对构建一个落地页，或者一个不需要路由的应用是有用的。

**有 pages 目录**：这时必须用 `<NuxtPage>` 来显示当前页面。

```vue [app/app.vue]
<template>
  <NuxtPage />
</template>
```

也可以把应用的公共结构直接写在 `app.vue` 里，官方给的用途是"包含页头或页脚这类全局元素"：

```vue [app/app.vue]
<template>
  <header>
    Header content
  </header>
  <NuxtPage />
  <footer>
    Footer content
  </footer>
</template>
```

这里有两条必须记住的限制：

- 官方提醒：**`app.vue` 是你的 Nuxt 应用的主组件，你往它里加的任何东西（JS 和 CSS）都会是全局的，并包含在每一个页面里。**
- 反过来，`app.vue` 本身是**可选的**：官方原话是如果你有 `app/pages/` 目录，`app.vue` 文件就是可选的，Nuxt 会自动包含一个默认的 `app.vue`，但你仍然可以加上自己的来自定义结构和内容。

## 第二层：组件不用 import

官方的定位很平实：大多数组件是可复用的界面碎片，比如按钮和菜单；在 Nuxt 里你可以在 `app/components/` 目录创建它们，它们会在全应用自动可用，**不需要显式导入**。

```vue [app/app.vue]
<template>
  <div>
    <h1>Welcome to the homepage</h1>
    <AppAlert>
      This is an auto-imported component.
    </AppAlert>
  </div>
</template>
```

```vue [app/components/AppAlert.vue]
<template>
  <span>
    <slot />
  </span>
</template>
```

注意上面这个例子里 `<AppAlert>` 是直接用的，没有 `import`、也没有注册。这条"自动可用"的边界（哪些目录会被扫、名字怎么生成、`global` 目录）单独有专篇讲，这一层你只要记住：**组件是这几层里唯一不需要交接手续的一层。**

## 第三层：页面 = 一个文件对应一个路由

官方定义：页面为每一个特定路由模式表示一个视图；`app/pages/` 目录里的**每个文件**都代表一个不同的路由并展示其内容。

启用它只有两个动作，官方原话是这样：创建一个 `app/pages/index.vue` 文件，并在 `app/app.vue` 里加上 `<NuxtPage />` 组件（**或者删掉 `app/app.vue` 使用默认入口**）；之后你就可以通过在 `app/pages/` 里新增文件来创建更多页面和它们对应的路由。

```vue [app/pages/index.vue]
<template>
  <div>
    <h1>Welcome to the homepage</h1>
    <AppAlert>
      This is an auto-imported component
    </AppAlert>
  </div>
</template>
```

```vue [app/pages/about.vue]
<template>
  <section>
    <p>This page will be displayed at the /about route.</p>
  </section>
</template>
```

"或者删掉 `app/app.vue`"这一步容易被误解，值得强调：默认入口本身就带页面出口，所以你**不需要**为了有路由而手写一个空的 `app.vue`。真正需要 `app.vue` 的时刻是你要在它里面加全局结构。

## 第四层：布局是包在页面外面的壳

官方定义：布局是包裹在页面外层、为多个页面包含公共 UI 的包装器，例如页头和页脚展示；**布局是使用 `<slot />` 组件来显示"页面"内容的 Vue 文件**；`app/layouts/default.vue` 会被默认使用；自定义布局可以作为页面元数据的一部分来设置。

```vue [app/app.vue]
<template>
  <div>
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>
```

```vue [app/layouts/default.vue]
<template>
  <div>
    <AppHeader />
    <slot />
    <AppFooter />
  </div>
</template>
```

`<slot />` 落点处的内容就是 `<NuxtPage />` 渲染出来的那一部分——也就是上面 index.vue、about.vue 的内容。

**但官方对这一层给了一句很明确的劝退**：如果你的应用里只有一个布局，它建议改用 `app/app.vue` 配合 `<NuxtPage />`。

对照第三层里那个 header/footer 写在 `app.vue` 的例子，你会发现两者渲染出来的结构是一样的。区别在于**多布局时才有必要动用 `app/layouts/`**（官方原话：当你的应用需要对不同页面有不同布局时，使用 `app/layouts/` 目录配合 `<NuxtLayout>` 组件，这样你可以定义多个布局并按页面应用它们）。单布局还坚持建一个 `layouts/default.vue`，等于多绕了一层没有任何收益的壳。

## 第五层：想动 HTML 本身，得走 Nitro 钩子

组件层再往外就是 HTML 模板了。官方的路径是：你可以添加一个注册钩子的 Nitro 插件，来获得对 HTML 模板的完全控制；`render:html` 钩子的回调函数让你在 HTML 发送给客户端**之前**改动它。

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

官方在这里同时挂了两个提醒：如果只需要修改 `<head>`，应该去看 SEO 与 meta 那部分；另外注释里点明 `render:html` 拿到的 `html` 是**HTML 模板的对象表示**，不是字符串——这决定了你能写的是 `html.head.push(...)` 这种结构化的操作。

## 我该放哪一层

| 你要做的事 | 放哪一层 |
| --- | --- |
| 一个按钮、一张卡片、一个菜单 | `app/components/`，直接用，不用 import |
| 一个新 URL 对应的内容 | `app/pages/` 里新加一个文件 |
| 每个页面都要出现的页头页脚（且只有一套） | 直接写在 `app/app.vue` 里，配合 `<NuxtPage />` |
| 不同页面用不同的外壳 | `app/layouts/` + 页面元数据指定布局 |
| 往每一页注入一段 JS 或 CSS | `app.vue`（官方提醒它全局生效）或配置项，视内容而定 |
| 改动最终 HTML 结构本身 | Nitro 插件 + `render:html` |
| 只想改 `<head>` 里的 meta | 别写钩子，走 SEO 与 meta 那条路 |

## 自查

1. 官方说 Nuxt 默认把 `app.vue` 当作什么？它为哪些路由渲染内容？
2. `app/pages/` 目录不存在时，Nuxt 少装了什么依赖？官方说这对哪类应用有用？
3. 有了 pages 目录，`app.vue` 里必须出现哪个组件？
4. 布局文件靠什么把页面内容显示出来？默认使用的是哪个文件？
5. 只有一个布局时官方的建议是什么？为什么？
6. 要改动发送给客户端之前的 HTML，官方给的路径是什么？`html` 参数是什么形态？

## 下一篇

091 展开 `<NuxtPage>`：它和 Vue Router 的 `<RouterView>` 是什么关系，为什么官方写着"必须用它、否则 `useRoute()` 可能返回错误的路径"，以及它内部到底套了哪几层组件。
