---
url: /posts/20f8fb623e5aa4885210023103895d87/
title: NuxtIsland 的参数、source 的风险与 useId 冲突
date: 2026-11-16T11:00:00+08:00
updated: 2026-11-16T11:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtIsland>` 的官方定位是"渲染一个**非交互式**组件，不带任何客户端 JS"——渲染岛屿组件时其内容是静态的，因此客户端不会下载任何 JS；而**改动岛屿组件的 props 会触发一次重新取回（refetch）以再次渲染它**；官方还明确"仅服务端的组件（server-only components）在底层使用 `<NuxtIsland>`"。它的 prop 只有五个：`name`（要渲染的组件名，`string`、**required**）、`lazy`（"让组件变为非阻塞"，`boolean`、默认 `false`）、`props`（要发送给被渲染组件的 props，`Record<string, any>`）、`source`（"要调用的远程源以渲染岛屿"，`string`）、`dangerouslyLoadClientComponents`（**从远程源加载客户端组件所必需**，`boolean`、默认 `false`）。两条 note 界定了远程岛屿：**远程岛屿需要你在 `nuxt.config` 里把 `experimental.componentIslands` 设为 `'local+remote'`**；默认情况下组件岛屿是从 `~/components/islands/` 目录扫描而来的，所以 `~/components/islands/MyIsland.vue` 可以用 `<NuxtIsland name="MyIsland" />` 渲染。而 `source` 这一格上挂着官方最重的一条警告：**"使用 `source` prop 从远程服务器渲染内容本质上（inherently）是危险的。当你指定一个远程 `source` 时，你完全信任那个服务器会提供安全的 HTML 内容，而这些内容会被直接渲染进你的应用。这与对外部内容使用 `v-html` 类似——远程服务器可以注入任何 HTML，包括潜在的恶意内容。只在你能完全信任并控制的服务器上使用 `source`。"** 官方还补充 `dangerouslyLoadClientComponents` 是**另一层**风险（是否也从远程源下载并执行客户端组件），即使它关着（默认值），你仍然在信任远程服务器的 HTML 输出。另有专门一节 Known Limitations 讲 `useId`：每个岛屿都在服务器上用自己的 Vue 应用渲染，所以 Vue 的 `useId` 计数器**对每个岛屿重新开始**，岛屿内生成的 id 可能与同页其他岛屿或应用其余部分生成的 id **冲突**；官方给的绕法是从服务端插件里基于 island context id 设置一个不同的 `idPrefix`，但两条残余问题写得很死——**相同 name、props 与 context 的两个岛屿共享一次服务端渲染与同一个 payload 条目，因此它们的 HTML（包括 `useId` 生成的 id）完全相同、作为前缀的岛屿 context id 也相同，结果是 DOM 里出现重复的 `id` 属性，可能破坏两个实例之间的 `aria-*` 引用与 `<label for>` 关联，而这种情况官方写明"目前没有绕法"**；以及**岛屿内的交互式组件（用 `nuxt-client` 属性加载的那个）里 `useId` 不可用**，因为它在岛屿的应用里服务端渲染、却由主客户端应用水合，服务端与客户端返回不同的值会造成水合不匹配。对外接口还有：插槽（如果岛屿里声明了就可以传，**每个插槽都是可交互的，因为提供它的是父组件**；`#fallback` 是保留插槽，用于"在岛屿加载之前（当组件是 lazy 时）或 `<NuxtIsland>` 取回组件失败时"渲染的内容）、ref 上的 `refresh()`（类型 `() => Promise<void>`，"通过重新取回来强制重新取回该服务端组件"）与 `error` 事件（类型 `unknown`，"当 `NuxtIsland` 未能取回新岛屿时发出"）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 岛屿
  - nuxtisland
  - 安全
  - useId

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇讲了为什么要有岛屿。这一篇看那个真正能干活的组件：`<NuxtIsland>` 只有五个 prop，但其中两个带官方明写的安全警告，还有一整节专门列限制。

## 官方定位与两条基本行为

> Nuxt 提供 `<NuxtIsland>` 组件，用来渲染一个**非交互式**组件，不携带任何客户端 JS。

配套两句：

| 官方原话 | 含义 |
| --- | --- |
| "在渲染一个岛屿组件时，该岛屿组件的内容是静态的，因此**客户端不会下载任何 JS**。" | 静态 = 没有交互能力 |
| "改动岛屿组件的 props 会**触发一次对该岛屿组件的重新取回**，以再次渲染它。" | 改 props 就是走网络 |
| （tip）"仅服务端的组件在底层使用 `<NuxtIsland>`。" | `.server.vue` 是它的自动包装 |

最后一条决定了取舍：**要用 `error` 事件和 `refresh()`，就只能直接写 `<NuxtIsland>`**——官方在限制清单里写明，通过自动生成的包装渲染的服务端组件**不暴露**加载与错误事件，需要 `error` 事件与 `refresh()` 方法就直接使用 `<NuxtIsland>`。

## 五个 prop

| prop | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `name` | 要渲染的组件名 | `string` | **required** |
| `lazy` | 让组件变为**非阻塞** | `boolean` | `false` |
| `props` | 要发送给被渲染组件的 props | `Record<string, any>` | —— |
| `source` | 要调用的远程源以渲染岛屿 | `string` | —— |
| `dangerouslyLoadClientComponents` | **从远程源加载客户端组件所必需** | `boolean` | `false` |

两条 note 把默认边界钉住：

- **默认情况下，组件岛屿是从 `~/components/islands/` 目录扫描的**。所以 `~/components/islands/MyIsland.vue` 可以用 `<NuxtIsland name="MyIsland" />` 渲染。
- **远程岛屿需要你在 `nuxt.config` 里把 `experimental.componentIslands` 设为 `'local+remote'`。**

`lazy` 这一格和 `#fallback` 是绑在一起的：官方对保留插槽的说明就是"指定在岛屿加载之前（**当组件是 lazy 时**）或 `NuxtIsland` 取回组件失败时要渲染的内容"。也就是说 `lazy` 打开后，那段等待期必须有东西顶着。

上一篇的导航代价结论在这里落地：**客户端导航时岛屿要发一次网络往返，除非你传 `lazy`（配 `#fallback`）让它非阻塞渲染。**

## `source`：官方用词最重的一格

这条警告值得整段引：

> 使用 `source` prop 从远程服务器渲染内容是**本质上危险的（inherently dangerous）**。当你指定一个远程 `source` 时，你完全信任那个服务器会提供安全的 HTML 内容，而这些内容将被直接渲染进你的应用。
>
> 这与对外部内容使用 `v-html` 类似——远程服务器可以注入任何 HTML，包括潜在的恶意内容。**只在你能完全信任并控制的服务器上使用 `source`。**
>
> `dangerouslyLoadClientComponents` prop 控制着**额外一层**风险：是否也从远程源下载并执行客户端组件。即使 `dangerouslyLoadClientComponents` 处于关闭状态（默认值），你仍然在信任远程服务器的 HTML 输出。

三段读出的判断顺序：

1. `source` 一旦指定，**HTML 的可信度就等于那台服务器的可信度**——官方直接拿 `v-html` 作类比。
2. `dangerouslyLoadClientComponents` 是**第二个开关**，而且它的后果是"下载并执行"远端组件，风险级别高于渲染远端 HTML。它的默认值是 `false`，而 prop 名字本身（`dangerously...`）就是官方的提示方式。
3. 关掉第二个开关**不等于安全**：官方明写"即使关着，你仍然在信任远程服务器的 HTML 输出"。

所以远程岛屿的适用条件很窄：**那台服务器得是你自己的**。

## props 与 context 走 GET 查询参数

官方 note 与上一篇那条同源，这里再列一次它的风险面：

> 组件 props 与 context 作为 **GET 查询参数**发送以实现缓存。查询参数可能出现在服务器访问日志、CDN 缓存与 HTTP `Referer` 头中。

**可缓存是收益，日志可见是代价。** 一句话结论：令牌、用户隐私、内部 ID 这类值不要放进岛屿 props 或 context。

另外，从服务端组件那一页还有一条同源的安全细则：**props 来自请求，就要当作不可信输入**——官方会拦掉最容易无意泄漏的那几种（岛屿未声明的顶层 `as`，以及开启 `vue.runtimeCompiler` 时 props 任意位置的 `template`）；未声明的 prop 会作为 attribute 透到岛屿的**单根元素**上，所以根是多态组件的岛屿要么设 `defineOptions({ inheritAttrs: false })`，要么把接受的 props 声明出来；**要根据调用方输入切换组件时，请通过一个已导入组件的白名单映射来分发，而不是把原始 prop 直接传进 `<component :is>`、`h()`、`resolveDynamicComponent()`**——因为一个字符串可以解析成任何已注册组件或 HTML 元素。

## 插槽：唯一"可交互"的入口

官方两句连着看：

- "如果岛屿里声明了，就可以向岛屿组件传插槽。**每个插槽都是可交互的，因为提供它的是父组件。**"
- 服务端组件页的解释："插槽内容由父组件提供，因此它属于主客户端应用，**是**可交互的（它被包在一个 `display: contents;` 的 `<div>` 里）。"

这一点很容易被误解成"岛屿里什么都不能点"：**岛屿本体不交互，但从外面传进去的插槽内容是完全正常的客户端组件。** 官方在配置层面还提供了第二个出口——`nuxt-client` 属性可以让岛屿里的某个本地 `.vue` 组件被主应用水合（需要 `selectiveClient`）。

保留插槽只有一个：`#fallback`。

## `useId`：一整节 Known Limitations

这是官方唯一单列成节的限制，原因写在机制里：

> 每个岛屿都在服务器上用它自己的 Vue 应用渲染，所以 Vue 的 `useId` 计数器**对每个岛屿重新开始**。在岛屿内部生成的 id 因此可能与同页其他岛屿、或应用其余部分生成的 id **冲突**。

官方给的绕法是在服务端插件里按 island context 的 id 设一个前缀：

```ts [plugins/island-id-prefix.server.ts]
export default defineNuxtPlugin((nuxtApp) => {
  const islandContext = nuxtApp.ssrContext?.islandContext
  if (islandContext) {
    nuxtApp.vueApp.config.idPrefix = `${islandContext.id}-v`
  }
})
```

然后两条残余问题，官方的写法没有任何余地：

- **完全相同的岛屿仍然共享同一批 id。** 两个实例如果 name、props 与 context 相同，就**共享一次服务端渲染（以及同一个 payload 条目）**，所以它们的 HTML——包括 `useId` 生成的那些 id——完全一样，而用作前缀的岛屿 context id 也一样。结果是 DOM 里出现**重复的 `id` 属性**，这可能在两个实例之间破坏 `aria-*` 引用与 `<label for>` 关联。**官方原话："目前这种情况没有绕法（There is currently no workaround for this case）。"**
- **岛屿内的交互式组件里 `useId` 不能用。** 用 `nuxt-client` 属性加载的组件在岛屿的应用里服务端渲染，却由主客户端应用水合，所以 `useId` 在服务端与客户端返回**不同的值**，造成水合不匹配。

实操结论：**岛屿里凡是靠 id 关联的东西（`<label for>`、`aria-labelledby`、`aria-describedby`）都要考虑重复风险**。两个一模一样的岛屿出现在同一页时，官方明说无解——要么让它们的 props 有差别，要么那部分别做成岛屿。

## ref 与事件

| 接口 | 官方说明 |
| --- | --- |
| `refresh()` | 类型 `() => Promise<void>`；"通过重新取回来强制重新取回该服务端组件。" |
| `error` 事件 | `error` 类型 `unknown`；"当 `NuxtIsland` 未能取回新岛屿时发出。" |

这两样正是自动包装（`.server.vue`）拿不到的那两个，也是"要错误处理就直写 `<NuxtIsland>`"的理由。

## 自查

1. `<NuxtIsland>` 的五个 prop 分别是什么？哪两个只在远程岛屿场景出现？
2. 渲染远程岛屿需要 `experimental.componentIslands` 设成什么？默认从哪个目录扫描岛屿？
3. 官方对 `source` 用了哪个形容词？它类比的是哪个指令？关掉 `dangerouslyLoadClientComponents` 是否等于安全？
4. `lazy` 为什么几乎必须和 `#fallback` 一起用？官方对后者的说明包含哪两种时机？
5. 岛屿里唯一可交互的内容是什么？配置层面还有哪个出口？
6. 为什么 `useId` 在岛屿里会撞？官方给的两种残余情况分别是什么？
7. 想要 `error` 事件与 `refresh()`，写法上有什么要求？

## 下一篇

108 讲 `useAppConfig` 与 `updateAppConfig`：为什么 app.config 的值会进客户端包、`updateAppConfig` 是**深度赋值**所以删不掉键，以及主题切换该在生命周期的哪两个时机做。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-island`
  - 原文（描述与首句）："Nuxt provides the \<NuxtIsland> component to render a non-interactive component without any client JS."；"When rendering an island component, the content of the island component is static, thus no JS is downloaded client-side."；"Changing the island component props triggers a refetch of the island component to re-render it again."
  - 原文（tip）："Server only components use `<NuxtIsland>` under the hood"
  - 原文（Props）："`name` : Name of the component to render."（`string`、**required**）；"`lazy`: Make the component non-blocking."（`boolean`、default `false`）；"`props`: Props to send to the component to render."（`Record<string, any>`）；"`source`: Remote source to call the island to render."（`string`）；"`dangerouslyLoadClientComponents`: Required to load client components from a remote source."（`boolean`、default `false`）
  - 原文（note）："Remote islands need `experimental.componentIslands` to be `'local+remote'` in your `nuxt.config`."
  - 原文（note）："By default, component islands are scanned from the `~/components/islands/` directory. So the `~/components/islands/MyIsland.vue` component could be rendered with `<NuxtIsland name="MyIsland" />`."
  - 原文（warning）："Using the `source` prop to render content from a remote server is inherently dangerous. When you specify a remote `source`, you are fully trusting that server to provide safe HTML content that will be rendered directly in your application. This is similar to using `v-html` with external content - the remote server can inject any HTML, including potentially malicious content. **Only use `source` with servers you fully trust and control.** The `dangerouslyLoadClientComponents` prop controls an additional layer of risk: whether to also download and execute client components from the remote source. Even with `dangerouslyLoadClientComponents` disabled (the default), you are still trusting the remote server's HTML output."
  - 原文（note）："Component props and context are sent as GET query parameters to enable caching. Query parameters may be visible in server access logs, CDN caches, and HTTP `Referer` headers."
  - 原文（Slots）："Slots can be passed to an island component if declared. Every slot is interactive since the parent component is the one providing it. Some slots are reserved to `NuxtIsland` for special cases. - `#fallback`: Specify the content to be rendered before the island loads (if the component is lazy) or if `NuxtIsland` fails to fetch the component."
  - 原文（Ref）："`refresh()`"，type `() => Promise<void>`，"force refetch the server component by refetching it."
  - 原文（Events）："`error`"，type `unknown`，"emitted when `NuxtIsland` fails to fetch the new island."
  - 原文（Known Limitations - `useId`）："Each island is rendered in its own Vue app on the server, so Vue's `useId` counter restarts for every island. Ids generated inside an island can therefore collide with ids generated by other islands on the same page, or by the rest of your app."；**Workaround** 示例（`plugins/island-id-prefix.server.ts`）：`const islandContext = nuxtApp.ssrContext?.islandContext`、`if (islandContext) { nuxtApp.vueApp.config.idPrefix = \`${islandContext.id}-v\` }`
  - 原文："**Identical islands still share the same ids.** Two instances of the same island rendered with the same name, props and context share a single server render (and payload entry), so their HTML — including any `useId`-generated ids — is identical, and the island context id used as prefix is the same. This results in duplicated `id` attributes in the DOM, which can break `aria-*` references and `<label for>` associations between the two instances. There is currently no workaround for this case."
  - 原文："**`useId` does not work in interactive components inside islands.** A component loaded with the `nuxt-client` attribute is server-rendered inside the island's app but hydrated by the main client app, so `useId` returns different values on the server and on the client, causing a hydration mismatch."
  - 原文 Source：`packages/nuxt/src/app/components/nuxt-island.ts`
- `https://nuxt.com/docs/4.x/guide/concepts/server-components`
  - 对照用（见 106）：插槽"belongs to the main client app and **is** interactive (it is wrapped in a `<div>` with `display: contents;`)"；"`<NuxtIsland>` reserves the `#fallback` slot to specify content rendered before the island loads (when `lazy` is set) or when fetching the island fails."；"islands block on a network round trip during navigation, unless you pass the `lazy` prop (with a `#fallback` slot) to render them non-blockingly"
  - 原文（安全细则 note）："Because props come from the request (URL query or body), treat them as untrusted input. Nuxt rejects the props most likely to leak through unintentionally: a top-level `as` that the island does not declare (an undeclared prop falls through as an attribute onto the island's root), and, with `vue.runtimeCompiler` enabled, a `template` anywhere in the props."；"Props a component does not declare fall through as attributes onto its single root element... Set `defineOptions({ inheritAttrs: false })` on such islands, or declare the props you accept."；"To switch components based on caller input, map a discriminator through an allowlist of imported components rather than passing the raw prop"
  - 原文（Current Limitations）："Server components rendered via the auto-generated wrapper do not expose load and error events; use `<NuxtIsland>` directly if you need its `error` event and `refresh()` method."
  - 原文（Selective Hydration）："you can hydrate individual components inside it by adding the `nuxt-client` attribute. This requires `experimental.componentIslands.selectiveClient` to be enabled."；"Setting `selectiveClient: 'deep'` additionally allows passing slots to `nuxt-client` components. Those slots are rendered on the server and are **not interactive** on the client."
