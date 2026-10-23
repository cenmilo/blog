---
url: /posts/1f990feb12449d57be27a47b2e8dd26b/
title: 117 - definePageMeta 是编译器宏：meta 被提到组件之外带来三条限制
date: 2026-11-20T09:00:00+08:00
updated: 2026-11-20T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：页面里被 Nuxt 读走的那部分内容，全都来自 `definePageMeta`，而它的实现方式决定了一堆"看起来能写、其实拿不到"的写法。官方先给了用途："你可能会想为应用里的每条路由定义元数据。**你可以用 `definePageMeta` 宏来做这件事，它在 `<script>` 与 `<script setup>` 里都能工作**"，写进去的数据"**可以在你应用的其余部分通过 `route.meta` 对象访问**"（官方示例：`definePageMeta({ title: 'My home page' })` 之后 `console.log(route.meta.title) // My home page`）。然后是嵌套时的合并规则：**"如果你使用嵌套路由，所有这些路由的页面元数据会被合并进一个单一的对象。"** 最关键的一段是宏的身份："**与 `defineEmits` 或 `defineProps` 很像，`definePageMeta` 是一个编译器宏。它会被编译掉，因此你无法在组件内部引用它。取而代之的是，传给它的元数据会被提升到组件之外。**"由此官方直接推出可用与不可用："**因此页面 meta 对象不能引用组件本身。不过它可以引用导入的绑定，以及局部定义的纯函数。**"，并挂了一条 warning："**务必不要引用任何响应式数据或会造成副作用的函数。这会导致意外的行为。**"官方的反例示例把踩法标注得最清楚：`const title = ref('')` 之后把 `title` 放进 `definePageMeta({ ..., title })`，注释原文是 `// do not do this, the ref will be hoisted out of the component`。特殊字段方面，`app/pages/` 那一节列了九项（`alias`、`keepalive`、`key`、`layout`、`layoutTransition` 与 `pageTransition`、`middleware`、`name`、`path`、`props`），其中 `middleware` 有一条合并口径："**它会与任何匹配的父/子路由里用到的所有其他中间件合并。**"；`keepalive` 则给出父路由保状态的专用写法 `<NuxtPage keepalive />` 与"可以在你的 nuxt.config 里为这个属性设置默认值"。自定义字段官方鼓励类型增强，示例是 `declare module '#app' { interface PageMeta { pageType?: string } }`，并且必须带上那句注释所说的事：**"当你增强一个类型时，务必确保导入了或导出了某些内容"**（示例末尾就是 `export {}`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - definepagemeta
  - 编译器宏
  - route-meta
  - 类型增强

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


`definePageMeta` 里写一个 `ref`、写一个用到组件变量的函数，页面照样跑，但值永远不对——因为这段东西在编译阶段就被**搬出组件**了。这一篇说清楚它是怎么被读走的。

## 用途与两种落点

官方定义：

> 你可能会想为应用里的每条路由定义元数据。**你可以用 `definePageMeta` 宏来做这件事，它在 `<script>` 与 `<script setup>` 里都能工作。**

```vue
<script setup lang="ts">
definePageMeta({
  title: 'My home page',
})
</script>
```

读回来的地方是 `route.meta`：**"这些数据可以在你应用的其余部分通过 `route.meta` 对象访问。"**

```vue
<script setup lang="ts">
const route = useRoute()

console.log(route.meta.title) // My home page
</script>
```

注意这一步的前提：`title` 不是 Nuxt 认识的字段，它就是你自己放进去的自定义字段——官方对这一类的说明是"**除了上面的属性之外，你也可以设置自定义元数据**"，并建议用类型增强来保证类型安全（下面会讲）。

嵌套还有一条合并规则要提前知道：**"如果你使用嵌套路由，所有这些路由的页面元数据会被合并进一个单一的对象。"** 也就是说 `route.meta` 拿到的是合并结果，不是当前这一层的。

## 它是编译器宏：三条后果

官方这段是这一篇的核心，逐句拆：

> **与 `defineEmits` 或 `defineProps` 很像，`definePageMeta` 是一个编译器宏。它会被编译掉，因此你无法在组件内部引用它。取而代之的是，传给它的元数据会被提升到组件之外。**
>
> **因此页面 meta 对象不能引用组件本身。不过它可以引用导入的绑定，以及局部定义的纯函数。**

由此得出三条限制：

| 限制 | 具体含义 |
| --- | --- |
| **不能在组件里引用它** | 它被编译掉，所以 `definePageMeta` 这个标识符在你的代码里根本不存在，别去 `import` 它、也别想读回自己写了什么 |
| **meta 不能引用组件本身** | 提升之后它就不在组件作用域里了 |
| **能引用的只有两类** | 导入的绑定（`import` 进来的东西）+ 局部定义的**纯函数** |

第三条里"纯函数"这个限定词是官方的用词，对应地，官方紧跟了一条 warning：

> **务必不要引用任何响应式数据或会造成副作用的函数。这会导致意外的行为。**

### 官方给的反例

```vue
<script setup lang="ts">
import { someData } from '~/utils/example'

function validateIdParam (route) {
  return route.params.id && !Number.isNaN(Number(route.params.id))
}

const title = ref('')

definePageMeta({
  validate: validateIdParam,
  someData,
  title, // do not do this, the ref will be hoisted out of the component
})
</script>
```

这段示例其实是"对照表"，三种写法各代表一类：

| 写法 | 官方态度 |
| --- | --- |
| `validate: validateIdParam` | ✔ 局部纯函数，官方把它当正例放在这里 |
| `someData`（来自 `~/utils/example` 的导入） | ✔ 导入的绑定 |
| `title`（`ref('')`） | ✘ 注释原文就是 **"do not do this, the ref will be hoisted out of the component"** |

最后一行的表现值得说清楚：它不会报错，而是**那个 ref 被原样提升到组件外面**——你后面 `title.value = 'x'` 改了，`route.meta.title` 也不会跟着变。这就是"意外的行为"的具体形态。

## 被 Nuxt 特殊对待的字段

官方在 `app/pages/` 那一节列了"特殊元数据"（Special Metadata）九项，并说明"**你当然可以为自己的用途随意定义元数据。但用 `definePageMeta` 定义的某些元数据有特定用途**"：

| 字段 | 官方的特殊用途说明 |
| --- | --- |
| `alias` | 从不同路径访问同一页面；字符串或字符串数组 |
| `keepalive` | 设 `true` 时 Nuxt 会自动把你的页面包进 Vue 的 `<KeepAlive>`；**父路由保状态请用 `<NuxtPage keepalive />`**；可在 `nuxt.config` 里设默认值 |
| `key` | 见 Child Route Keys（092） |
| `layout` | `false`（禁用任何布局）、字符串，或 ref/computed（要某种程度上的响应式） |
| `layoutTransition` / `pageTransition` | 给包裹页面与布局的 `<transition>` 定义属性，或传 `false` 关掉该路由的 transition 包装；可设默认值 |
| `middleware` | 字符串、函数（匿名/内联，遵循全局前置守卫模式），或字符串/函数数组；**"它会与任何匹配的父/子路由里用到的所有其他中间件合并"** |
| `name` | 为这条路由定义名字 |
| `path` | 定义 path 匹配器（比文件名更复杂的模式） |
| `props` | 允许把路由 `params` 作为 props 传给页面组件 |

**两处清单不完全一致，要以更全的那份为准。** `definePageMeta` 的参数表里还有 `validate`、`redirect`、`scrollToTop`、`viewTransition`、`groups`，而 `app/pages/` 那一节的"特殊元数据"列表只到 `props`。写这些字段时查参数表更可靠。

顺带一条与 `scrollToTop` 有关的口径（它常被误认成"重渲染才有滚动"）：官方的说法是"**导航独立于渲染，所以即使页面不重新渲染（例如使用了固定 `key` 时），滚动行为也总会触发。在这种情形下设 `scrollToTop: false` 来禁用滚动。**"——这正是 116 里那个 `key` 控制重建、与滚动无关的连线。

## 自定义字段的类型增强

官方给了完整做法：

> 如果你为页面添加自定义元数据，你可能希望以类型安全的方式来做。可以增强 `definePageMeta` 所接受对象的类型：

```ts [index.d.ts]
declare module '#app' {
  interface PageMeta {
    pageType?: string
  }
}

// It is always important to ensure you import/export something when augmenting a type
export {}
```

两个细节缺一不可：

| 细节 | 原因 |
| --- | --- |
| 增强的是 `#app` 里的 `PageMeta` | `route.meta` 的类型来自这里（105 讲过 `#app` 这个说明符） |
| 文件必须有 `export {}` | 官方注释："当你增强一个类型时，务必确保导入了或导出了某些内容"——没有导入/导出，这个文件会被当作脚本，`declare module` 就不生效 |

顺带一个熟悉的模式：`updateAppConfig` 那篇（108）里增强 app config 类型也是同一句要求，而那里增强的是 `nuxt/schema` 里的接口——**别把两个模块名写混**：路由元数据走 `#app` 的 `PageMeta`。

## 自查

1. `definePageMeta` 能在 `<script>` 和 `<script setup>` 里用吗？写进去的自定义字段从哪里读回来？
2. 官方把它类比成哪两个 Vue 宏？"被编译掉"和"被提升到组件之外"分别意味着什么？
3. meta 对象里可以引用哪两类东西？不可以引用什么？官方的 warning 原文怎么说？
4. 把 `ref` 放进 `definePageMeta` 会怎样？官方示例里那句注释是什么？
5. `middleware` 字段在父子路由之间是什么关系？`keepalive` 想保住父路由状态该写在哪？
6. 自定义元数据要做类型增强，应该声明哪个模块里的哪个接口？为什么必须写 `export {}`？

## 下一篇

118 讲过渡配置：`app.pageTransition` 与 `app.layoutTransition` 两个全局默认、CSS 类名四件套怎么和 `name` 对上，以及"同时换布局时页面过渡不会运行"这条官方提醒。
