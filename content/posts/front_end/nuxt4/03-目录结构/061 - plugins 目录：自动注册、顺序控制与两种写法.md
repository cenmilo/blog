---
url: /posts/011d93307c010507397017c618d4f51d/
title: 061 - plugins 目录：自动注册、顺序控制与两种写法
date: 2026-11-01T10:00:00+08:00
updated: 2026-11-01T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：Nuxt 会自动读取 app/plugins/ 里的文件并在 Vue 应用创建时加载它们，官方用 note 明确"里面的所有插件都会自动注册，你不需要额外把它们加进 nuxt.config"；文件名可以用 .server 或 .client 后缀只在服务端或客户端加载。扫描规则与中间件一致：只有目录顶层文件（或子目录里的 index 文件）会被自动注册，官方给的树里 foo.ts 被扫描、bar/baz.ts 与 bar/foz.vue 不被扫描、bar/index.ts 目前会被扫描但已标记 deprecated，要注册子目录里的插件请改用 nuxt.config 的 plugins 选项。插件的唯一入参是 nuxtApp；对象式写法可以声明 name、enforce（pre 或 post）、setup、hooks 与 env.islands，而官方警告：对象式的属性会被静态分析以产出更优化的构建，所以不要在运行时定义它们，例如 enforce: import.meta.server ? 'pre' : 'post' 会让 Nuxt 将来对你插件的优化失效。加载策略上默认是顺序加载，parallel: true 让 Nuxt 不等它执行完就加载下一个，dependsOn 数组则让一个插件等指定的插件跑完再运行。提供 helper 用 return { provide: { hello } }，但官方用 important 建议优先用 composable 以避免污染全局命名空间并保持主入口包小，且用 warning 指出如果插件 provide 的是 ref 或 computed，它在组件 template 里不会被解包。

categories:
  - 前端开发

tags:
  - nuxt4
  - 插件
  - 注册顺序
  - provide
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


009 我说过"插件是挂在应用创建时刻的东西"。这一篇讲它的目录侧规则：**谁会被自动注册、顺序怎么控制、两种写法各自的代价，以及官方那条"能别用 provide 就别用"的建议。**

## 两条 note：自动注册、`.client`/`.server` 后缀

> Nuxt 会自动读取 `app/plugins/` 目录里的文件，**并在 Vue 应用创建时加载它们。**
>
> （note）**里面的所有插件都会自动注册，你不需要额外把它们加进 `nuxt.config`。**
>
> （note）**你可以在文件名里使用 `.server` 或 `.client` 后缀，让一个插件只在服务端或客户端加载。**

**第一条解决了一个很常见的多余动作：往 `nuxt.config` 的 `plugins` 数组里手写一遍已经放在 `app/plugins/` 的文件。不必。**

第二条是这一层最实用的开关——013 讲"同一份代码跑两个地方"时说过插件可服务端/客户端/两侧，这里给的就是那个后缀写法（下面"Vue 指令"一节官方还会用它当解法）。

## 扫描规则：那棵树里三个"不被扫描"

**这条与 060 讲的中间件规则是同一句**（只有顶层文件与子目录 index 会被注册），但官方在这里给的树更直白，我照抄：

```bash [Directory structure]
-| plugins/
---| foo.ts      // scanned
---| bar/
-----| baz.ts    // not scanned
-----| foz.vue   // not scanned
-----| index.ts  // currently scanned but deprecated
```

> 只有 `foo.ts` 与 `bar/index.ts` 会被注册。

**注意最后那行注释：`currently scanned but deprecated`。** 也就是说**"子目录 + index.ts"这种组织方式官方已经标了废弃**——它能工作，但不该是新写的选择。

要注册子目录里的插件，官方给的替代方案是显式列出：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  plugins: [
    '~/plugins/bar/baz',
    '~/plugins/bar/foz',
  ],
})
```

**于是插件的组织策略很清楚：要么放顶层（自动注册），要么放进子目录并在 `nuxt.config` 里点名。用 `bar/index.ts` 钻空子是 deprecated 路径。**

## 两种写法：函数式与对象式

> 传给插件的唯一参数是 `nuxtApp`。

```ts [plugins/hello.ts]
export default defineNuxtPlugin((nuxtApp) => {
  // Doing something with nuxtApp
})
```

对象式给的是更多控制点：

```ts [plugins/hello.ts]
export default defineNuxtPlugin({
  name: 'my-plugin',
  enforce: 'pre', // or 'post'
  async setup (nuxtApp) {
    // this is the equivalent of a normal functional plugin
  },
  hooks: {
    // You can directly register Nuxt app runtime hooks here
    'app:created' () {
      const nuxtApp = useNuxtApp()
      // do something in the hook
    },
  },
  env: {
    // Set this value to `false` if you don't want the plugin to run when rendering server-only or island components.
    islands: true,
  },
})
```

把这份对象里五个键各自的用途分清（全都在这一页的原文里）：

| 键 | 作用 |
| --- | --- |
| `name` | 插件名，`dependsOn` 用它引用 |
| `enforce` | `'pre'` 或 `'post'`，插在其它插件之前或之后 |
| `setup` | 等价于函数式插件本体 |
| `hooks` | **直接在这里注册 Nuxt 应用运行时钩子** |
| `env.islands` | 设为 `false` 时，渲染 server-only 或 island 组件就不跑这个插件 |

**然后是这一节最该记住的警告：**

> 如果你使用对象语法，**这些属性会被静态分析以产出更优化的构建。所以你不应该在运行时定义它们。**
>
> 例如，**`enforce: import.meta.server ? 'pre' : 'post'` 会让 Nuxt 将来能对你插件做的任何优化失效。**
>
> Nuxt 在使用对象语法时会**静态预加载任何钩子监听器**，这让你可以定义钩子而不必担心插件注册顺序。

**这三句话里有一个明确的取舍**：对象式换来的是"钩子不必操心注册顺序"，代价是**属性必须静态可写**。**`enforce`、`name`、`parallel`、`dependsOn` 这几个键里不要放三元、不要放变量、不要放函数调用。**

（`env.islands` 那条注释提到的 server-only / island 组件，008 已经认过名字，机制在后续篇里展开。）

## 顺序：文件名字母序编号

> 你可以通过给文件名加上"字母序"的数字前缀，来控制插件被注册的顺序。

```bash [Directory structure]
plugins/
 | - 01.myPlugin.ts
 | - 02.myOtherPlugin.ts
```

> 在这个例子里，**`02.myOtherPlugin.ts` 能访问 `01.myPlugin.ts` 注入的任何东西。** 这在"一个插件依赖另一个插件"的情形下很有用。

官方在这里重复了那条我 060 抄过的坑，措辞完全一致：

> 如果你是第一次接触"字母序编号"，请记住**文件名是作为字符串排序的，而不是作为数值**。例如 `10.myPlugin.ts` 会排在 `2.myOtherPlugin.ts` 之前。这就是为什么例子给个位数加了 `0`。

**同一条规则出现在中间件与插件两处，说明这是框架侧的通用约定**：编号一律补零。

## 加载策略：默认顺序，`parallel` 不等，`dependsOn` 要等

> 默认情况下，Nuxt **顺序加载**插件。你可以把一个插件定义为 `parallel`，这样 Nuxt **不会等到它执行结束**才加载下一个插件。

```ts [plugins/my-plugin.ts]
export default defineNuxtPlugin({
  name: 'my-plugin',
  parallel: true,
  async setup (nuxtApp) {
    // the next plugin will be executed immediately
  },
})
```

> 如果一个插件需要先等另一个插件，你可以把那个插件的名字加进 `dependsOn` 数组。

```ts [plugins/depending-on-my-plugin.ts]
export default defineNuxtPlugin({
  name: 'depends-on-my-plugin',
  dependsOn: ['my-plugin'],
  async setup (nuxtApp) {
    // this plugin will wait for the end of `my-plugin`'s execution before it runs
  },
})
```

**这两个键是一对**：`parallel` 放开顺序，`dependsOn` 再把必要的依赖关系钉回去。**注意两者都只存在于对象式写法里**（上一节的"属性必须静态"警告因此直接适用于它们）。

## 插件里能用组合式函数，但有两处不行

> 你可以在 Nuxt 插件里使用 composables 以及 utils：

```ts [app/plugins/hello.ts]
export default defineNuxtPlugin((nuxtApp) => {
  const foo = useFoo()
})
```

然后两条 important 级别的限制：

> **如果一个组合式函数依赖于稍后才注册的另一个插件，它可能不生效。**
> 插件是按顺序、在其它一切之前被调用的。你可能用到了一个依赖尚未被调用的插件的组合式函数。

> **如果一个组合式函数依赖 Vue.js 的生命周期，它不会生效。**
> 通常 Vue.js 组合式函数绑定到当前组件实例，而**插件只绑定到 `nuxtApp` 实例**。

**第二条尤其关键，它是"插件与组件的本质区别"：** 插件不在组件树里，**所以 `onMounted` 那一类依赖组件实例的东西在这儿没有落点**。

**两条合起来给的判据很清晰：写插件时，先问这个组合式函数依赖哪个插件、依赖不依赖组件实例。**

## 提供 helper：官方的建议是"尽量别用"

> 如果你想在一个 helper 提供到 `NuxtApp` 实例上，**在插件里从一个 `provide` 键下返回它。**

```ts [plugins/hello.ts]
export default defineNuxtPlugin(() => {
  return {
    provide: {
      hello: (msg: string) => `Hello ${msg}!`,
    },
  }
})
```

```ts [plugins/hello-object-syntax.ts]
export default defineNuxtPlugin({
  name: 'hello',
  setup () {
    return {
      provide: {
        hello: (msg: string) => `Hello ${msg}!`,
      },
    }
  },
})
```

用法在组件里：

```vue [app/components/Hello.vue]
<script setup lang="ts">
// alternatively, you can also use it here
const { $hello } = useNuxtApp()
</script>

<template>
  <div>
    {{ $hello('world') }}
  </div>
</template>
```

**注意官方把前缀写成了 `$hello`**——provide 出来的键带 `$` 前访问。

然后是这一节真正的建议（important）：

> **我们强烈推荐改用 composables，而不是提供 helper，以避免污染全局命名空间并保持你的主入口包体更小。**

**这条把"该把 helper 放哪儿"直接给了答案：放 `app/composables/`（063 讲它的自动导入规则），而不是插件的 `provide`。**

还有一个技术坑，官方用 warning 写着：

> **如果你的插件 provide 了一个 `ref` 或 `computed`，它在组件的 `<template>` 里不会被解包。**
>
> 这是因为 Vue 处理"不是模板顶层的 ref"的方式所致。

**后果很具体：模板里得写 `$count.value` 而不是 `$count`**（我按这句 warning 给的写法推论；官方在此处只指出"不会被解包"并指向 Vue 文档那一节）。

## 类型：返回的 helper 自动定型

> **如果你从插件里返回你的 helper，它们会被自动定型**；你会在 `useNuxtApp()` 的返回值里以及在你的模板里看到它们的类型。

> （note）如果你需要在另一个插件**内部**使用一个被提供的 helper，你可以调用 `useNuxtApp()` 拿到带类型的那个版本。**但通常这应当避免，除非你确定插件之间的顺序。**

需要手写时用两份模块增强（**注意是 `#app` 与 `vue` 两个模块**）：

```ts [index.d.ts]
declare module '#app' {
  interface NuxtApp {
    $hello (msg: string): string
  }
}

declare module 'vue' {
  interface ComponentCustomProperties {
    $hello (msg: string): string
  }
}

export {}
```

**为什么两处都要写？** 按上面那句"你会在 `useNuxtApp()` 返回值里**以及在你的模板里**看到类型"——**前者对应 `NuxtApp` 接口，后者对应 Vue 的 `ComponentCustomProperties`**。（这层对应关系是我对上述两处代码与那句话的读法。）

`export {}` 那一行 055 也出现过：**增强类型时必须 import/export 点东西。**

## 用 Vue 插件：`nuxtApp.vueApp.use()`

官方以 `vue-gtag-next` 为例，第一步是装依赖（五种包管理器都给）：

```bash [pnpm]
pnpm add -D vue-gtag-next
```

然后这个插件文件的名字很讲究——**带了 `.client` 后缀**：

```ts [app/plugins/vue-gtag.client.ts]
import VueGtag, { trackRouter } from 'vue-gtag-next'

export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.use(VueGtag, {
    property: {
      id: 'GA_MEASUREMENT_ID',
    },
  })
  trackRouter(useRouter())
})
```

**`nuxtApp.vueApp.use(...)` 就是把 Vue 应用实例交给第三方 Vue 插件去 `use`。** 而官方选的文件名是 `vue-gtag.client.ts`——**分析类库只在浏览器里有意义，所以用后缀把它锁在客户端**（这条性质对应本节开头那条 note）。

## 自定义指令：必须两端注册

> 类似地，你可以在插件里注册一个自定义 Vue 指令。

```ts [plugins/my-directive.ts]
export default defineNuxtPlugin((nuxtApp) => {
  nuxtApp.vueApp.directive('focus', {
    mounted (el) {
      el.focus()
    },
    getSSRProps (binding, vnode) {
      // you can provide SSR-specific props here
      return {}
    },
  })
})
```

**注意示例里带了 `getSSRProps`**——服务端渲染时指令要提供的属性。官方随后用 warning 把要求写死：

> 如果你注册一个 Vue 指令，**你*必须*在客户端与服务端两侧都注册它**，除非你只在其中一侧渲染时使用它。**如果这个指令只在客户端有意义，你总可以把它移到 `~/plugins/my-directive.client.ts`，并在 `~/plugins/my-directive.server.ts` 里为服务端提供一个"桩"指令。**

**这就是这一节最可执行的做法：一个纯客户端指令 = `.client.ts` 写实现 + `.server.ts` 写空壳。** 少了那个桩，服务端渲染就会出问题。

## 自查

- [ ] 放在 `app/plugins/` 里的文件要不要再写进 `nuxt.config`？
- [ ] `bar/baz.ts` 会被自动注册吗？`bar/index.ts` 呢？官方给它标了什么？
- [ ] `enforce: import.meta.server ? 'pre' : 'post'` 有什么问题？
- [ ] `parallel` 与 `dependsOn` 各自解决什么？它们只能写在哪种写法里？
- [ ] 插件里为什么不能用依赖 Vue 生命周期的组合式函数？
- [ ] 官方更推荐用 provide 还是 composable？理由写了哪两条？
- [ ] 一个纯客户端的 Vue 指令，官方给的落地文件结构是什么？

## 下一篇

062 讲 `components/` 目录：组件名是怎么从路径与前缀生成的、`pathPrefix` 与 `prefix` 各自改哪一段、`(group)` 目录名的含义，以及用 `addComponent` 手动注册一个组件。

## 本文依据

- `directory-structure/app/plugins`（标题：Nuxt 有插件系统以便在 Vue 应用创建时使用 Vue 插件及更多内容；Nuxt 自动读取 app/plugins/ 里的文件并在 Vue 应用创建时加载；note 里面所有插件自动注册无需额外加入 nuxt.config；note 文件名可用 .server 或 .client 后缀只在服务端或客户端加载；Registered Plugins：只有目录顶层文件（或任意子目录中的 index 文件）会被自动注册，示例树 foo.ts scanned、bar/baz.ts not scanned、bar/foz.vue not scanned、bar/index.ts currently scanned but deprecated，只有 foo.ts 与 bar/index.ts 会被注册，要在子目录添加插件可用 nuxt.config.ts 的 plugins 选项（'~/plugins/bar/baz'、'~/plugins/bar/foz'）；Creating Plugins：传给插件的唯一参数是 nuxtApp，示例 defineNuxtPlugin((nuxtApp)=>{})；Object Syntax Plugins：更高级用例可用对象式，示例含 name/enforce 'pre'|'post'/async setup/hooks 内直接注册 Nuxt app runtime hooks（'app:created'）/env.islands（设为 false 则渲染 server-only 或 island 组件时不运行该插件）；note 使用对象语法时属性会被静态分析以产出更优化的构建故不应在运行时定义它们，例如 enforce: import.meta.server ? 'pre' : 'post' 会让 Nuxt 将来对你插件的任何优化失效，Nuxt 使用对象语法时会静态预加载任何钩子监听器使你可以定义钩子而不必担心插件注册顺序；Registration Order：可用"字母序"数字前缀控制注册顺序（01.myPlugin.ts、02.myOtherPlugin.ts），后者能访问前者注入的任何东西，适合一个插件依赖另一个插件的情形，note 文件名按字符串排序而非数值故 10.myPlugin.ts 排在 2.myOtherPlugin.ts 前这是给个位数加 0 的原因；Loading Strategy：默认 Nuxt 顺序加载插件、可把插件定义为 parallel 使 Nuxt 不等其执行结束就加载下一个（示例 name+parallel:true+注释 the next plugin will be executed immediately）、若插件需先等另一个插件可把其名字加入 dependsOn 数组（示例 dependsOn:['my-plugin'] 与注释 this plugin will wait for the end of my-plugin's execution before it runs）；Using Composables：插件内可使用 composables 与 utils 并给出 useFoo() 示例，important 两条限制——若组合式函数依赖稍后注册的另一个插件可能不生效（插件按顺序、在其它一切之前被调用）；若依赖 Vue.js 生命周期不会生效（通常 Vue 组合式函数绑定当前组件实例而插件只绑定 nuxtApp 实例）；Providing Helpers：要把 helper 提供到 NuxtApp 实例上需在插件里从 provide 键下返回，函数式与对象式两种示例均为 provide:{hello:(msg:string)=>`Hello ${msg}!`}，组件里 const { $hello } = useNuxtApp() 与模板 $hello('world')，important 强烈建议使用 composables 而非提供 helper 以避免污染全局命名空间并保持主入口包小，warning 若插件 provide 了 ref 或 computed 它在组件 template 中不会被解包（因为 Vue 处理非模板顶层 ref 的方式）；Typing Plugins：返回的 helper 会被自动定型并可在 useNuxtApp() 返回值与模板中看到类型，note 若需在另一个插件内部使用被提供的 helper 可调用 useNuxtApp() 取带类型版本但通常应避免除非确定插件顺序，进阶可用 declare module '#app' { interface NuxtApp { $hello(msg:string):string } } 与 declare module 'vue' { interface ComponentCustomProperties { $hello(msg:string):string } } 加 export {}；Vue Plugins：要用 Vue 插件（例如 vue-gtag-next 添加 Google Analytics 标签）可用一个 Nuxt 插件完成，先按五种包管理器安装依赖（pnpm add -D vue-gtag-next 等），再创建 app/plugins/vue-gtag.client.ts 并在 defineNuxtPlugin 里 nuxtApp.vueApp.use(VueGtag,{property:{id}}) 与 trackRouter(useRouter())；Vue Directives：可在插件里注册自定义 Vue 指令，示例 nuxtApp.vueApp.directive('focus',{mounted(el){el.focus()},getSSRProps(binding,vnode){return {}}})，warning 注册 Vue 指令必须客户端与服务端两侧都注册除非只在其中一侧渲染时使用，若指令只在客户端有意义可移到 ~/plugins/my-directive.client.ts 并在 ~/plugins/my-directive.server.ts 为服务端提供一个"桩"指令）
