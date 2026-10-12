---
url: /posts/c21ae105a1bf2591d7d5ba3e1b87cbad/
title: useAppConfig 与 updateAppConfig：主题值放这里但别放密钥
date: 2026-11-17T09:00:00+08:00
updated: 2026-11-17T09:00:00+08:00
author: cmdragon

summary:
  摘要：`app.config.ts` 的官方定位是"用 App Config 文件在你的应用内暴露**响应式**配置"，而那句定义里同时给出了它的三个更新时机——Nuxt 提供一个 `app/app.config.ts` 配置文件来在应用内暴露响应式配置，并**能够在生命周期内更新它、或使用一个 Nuxt 插件更新它、还能用 HMR（hot-module-replacement）编辑它**。文件本身可以有 `.ts`、`.js` 或 `.mjs` 三种扩展名；配了自定义 `srcDir` 时，官方要求把 `app.config` 文件放在那个新 `srcDir` 路径的**根部**。两条硬性提醒必须背下来：官方 caution 原文是"**不要把任何密钥值放进 `app.config` 文件。它会暴露到用户的客户端包里。**"；而读取侧的官方承诺是"我们现在可以在**服务端渲染页面时以及在浏览器里**普遍访问 `theme`"，用的就是 `useAppConfig()`（官方定义："访问项目中定义的响应式 app config"）。改动侧是 `updateAppConfig`，官方定义为"在运行时更新 App Config"，并有一条决定性的 note："**使用深度赋值（deep assignment）来更新 `app.config`。已有的（嵌套）属性会被保留。**"——这既是它能用来改主题色的原因（只传 `theme.primaryColor` 不会丢掉 `theme` 里其他键），也是它删不掉任何键的原因（它只叠加，不覆盖整个对象）。官方示例还显示这个工具是从 `#imports` 显式导入的（`import { updateAppConfig, useAppConfig } from '#imports'`）。类型侧官方分三种情况：Nuxt 会尝试自动从你提供的 app config 生成 TypeScript 接口；`AppConfigInput` 供模块作者声明"设置 app config 时合法的输入选项"，且**不影响 `useAppConfig()` 的类型**；要改 `useAppConfig()` 的返回类型则要扩展 `AppConfig`，但**在应用代码之外（服务端路由、`shared/` 目录以及 `nuxt.config`）`useAppConfig()` 返回的是 `SharedAppConfig`，所以要改扩展那个接口**，并且官方 warning 提醒"给 `AppConfig` 加类型时要小心，因为你会**覆盖** Nuxt 从你实际定义的 app config 推断出的类型"。

categories:
  - 前端开发

tags:
  - nuxt4
  - appconfig
  - 主题
  - 类型
  - 响应式

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


主题色、默认语言、某功能开关这类"配置里但人人都能看到"的值，放哪儿？`app.config.ts` 就是这一格。但它和 `runtimeConfig` 长得太像，放错的后果是密钥进包或者配置读不到。这一篇把它的所有接口过一遍。

## 官方定义里藏着三个更新时机

> Nuxt 提供一个 `app/app.config.ts` 配置文件，用来在你的应用内暴露**响应式配置**，并且能够在**生命周期内**、或**使用一个 Nuxt 插件**来在运行时更新它，还能用 **HMR（hot-module-replacement）** 编辑它。

这句话实际给了三条更新路径：

| 时机 | 对应做法 |
| --- | --- |
| 生命周期内 | 组件 `<script setup>` 里调用 `updateAppConfig(...)` |
| Nuxt 插件里 | 插件内调用同一工具（061 讲的插件注册时机） |
| 开发期编辑 | 改 `app.config.ts` 文件本身，走 HMR |

另外两条文件级规则：

- 扩展名官方给了三种："它可以带有 `.ts`、`.js` 或 `.mjs` 扩展名。"
- 位置：**"当配置了自定义 `srcDir` 时，请确保把 `app.config` 文件放在新的 `srcDir` 路径的根部。"**（046、079 里的同类结论）

## 第一条铁律：不放密钥

官方的 caution 只有一句，但它是安全边界：

> **不要把任何密钥值放进 `app.config` 文件。它会暴露到用户的客户端包里。**

原因就在读取侧——官方明确这个值是**两侧都能拿到**的："我们现在可以在服务端渲染页面时以及在浏览器里普遍访问 `theme`。"

```ts [app/app.config.ts]
export default defineAppConfig({
  theme: {
    primaryColor: '#ababab',
  },
})
```

```vue [app/pages/index.vue]
<script setup lang="ts">
const appConfig = useAppConfig()

console.log(appConfig.theme)
</script>
```

`useAppConfig` 的官方定义就一行："访问项目中定义的响应式 app config。"

### 与 `runtimeConfig` 的分界

两个都像配置，判据完全不同：

| | `app.config.ts` | `runtimeConfig` |
| --- | --- | --- |
| 官方定位 | 应用内的响应式配置 | 传递动态配置与环境变量到 Nuxt 应用上下文 |
| 可见性 | **会进客户端包**（故禁密钥） | "这个对象的值**只能从服务端**通过 `useRuntimeConfig` 访问。它主要应当存放**不暴露给前端的私有**配置，例如你的 API 密钥引用。`public` 与 `app` 之下的任何内容也会暴露给前端。" |
| 能否被环境变量覆盖 | 官方没这机制 | "值会在运行时被匹配的环境变量自动替换，例如设置环境变量 `NUXT_API_KEY=my-api-key NUXT_PUBLIC_BASE_URL=/foo/` 会覆盖下面示例中的两个值。" |
| 典型内容 | 主题色、UI 默认值 | 令牌、第三方密钥、公开 baseURL |

**一句话选边：能被用户在浏览器里看到且不该改的 → `app.config.ts`；不该被看到的 → `runtimeConfig`（且不在 `public`/`app` 之下）。**

## `updateAppConfig`：深度赋值意味着只能叠加、删不掉

官方定义为"在运行时更新 App Config"，紧跟着一条决定行为的 note：

> **使用深度赋值（deep assignment）来更新 `app.config`。已有的（嵌套）属性会被保留。**

官方的完整示例：

```js
import { updateAppConfig, useAppConfig } from '#imports'

const appConfig = useAppConfig() // { foo: 'bar' }

const newAppConfig = { foo: 'baz' }
updateAppConfig(newAppConfig)

console.log(appConfig) // { foo: 'baz' }
```

从这段读出三件事：

1. **同一个对象被就地更新**——`appConfig` 是同一个引用，`updateAppConfig` 之后它自己就变了（这也是它"响应式"的落点，主题切换因此不需要重新取一次）。
2. **同层的键会被覆盖，其它键保留。** `foo` 从 `bar` 变 `baz`，而示例里没写的那些键不受影响。
3. **删不掉键。** 深度赋值只叠加：传 `{ theme: { primaryColor: 'x' } }` 不会把 `theme` 里别的键抹掉，也不会把 `theme` 整个替换成新对象；想要"少了某个键"的效果，`updateAppConfig` 不是那个工具。

顺带注意那行 import——官方示例显式写 `from '#imports'`（105 讲过这个别名）。

### 切主题的标准写法

以官方给的 `theme.primaryColor` 为例，走 `updateAppConfig` 是最贴合文档的做法，而且**深度赋值正好保证你不会丢掉 `theme` 的其他键**：

```ts
updateAppConfig({
  theme: {
    primaryColor: '#1e40af',
  },
})
```

因为拿到的是同一个响应式对象，模板里读 `appConfig.theme.primaryColor` 的地方会随之更新——这三条（响应式、就地更新、保留嵌套）合起来就是它适合做主题的原因。

## 类型：三种情况各扩展不同接口

官方先给了默认行为：**"Nuxt 会尝试自动从你提供的 app config 生成一个 TypeScript 接口，所以你不需要自己给它加类型。"**

要自己管的时候，三种情况别混：

| 你要做什么 | 扩展谁 | 官方说明 |
| --- | --- | --- |
| 模块作者声明"合法的输入" | `AppConfigInput` | "`AppConfigInput` 可能被模块作者使用，用来声明设置 app config 时合法的*输入*选项。**这不会影响 `useAppConfig()` 的类型。**" |
| 应用代码里 `useAppConfig()` 的返回类型 | `AppConfig` | "如果你想给调用 `useAppConfig()` 的结果加类型，那就扩展 `AppConfig`。" |
| **应用代码之外** | `SharedAppConfig` | "**在应用代码之外（服务端路由、`shared/` 目录以及 `nuxt.config`），`useAppConfig()` 返回 `SharedAppConfig`，所以请改为扩展那个接口。**" |

第三行是新手最容易撞的：在服务端路由里 `useAppConfig()` 看到的类型和应用里不是同一个接口，因为官方按上下文区分了返回值类型。

还有一条官方 warning，写得很直白：

> 给 `AppConfig` 加类型时要小心，因为你会**覆盖** Nuxt 从你实际定义的 app config 推断出的类型。

官方示例里那一行注释就是这个覆盖的说明：`// This will entirely replace the existing inferred 'theme' property`。

两份声明文件都带同一条要求（官方注释原文）：`// It is always important to ensure you import/export something when augmenting a type`，示例结尾就是 `export {}`。

```ts [index.d.ts]
declare module 'nuxt/schema' {
  interface AppConfig {
    // This will entirely replace the existing inferred `theme` property
    theme: {
      // You might want to type this value to add more specific types than Nuxt can infer,
      // such as string literal types
      primaryColor?: 'red' | 'blue'
    }
  }
}

// It is always important to ensure you import/export something when augmenting a type
export {}
```

## 多层项目里怎么合：Function Merger

官方说明：**"Nuxt 为应用**各图层**里的 `AppConfig` 使用一种自定义合并策略。该策略通过一个 Function Merger 实现，它允许为 `app.config` 里每一个值为数组的键定义自定义合并策略。"** 并给了一条限制：**"函数合并器只能用在被扩展的图层里，不能用在项目主 `app.config` 里。"**

这条只在多图层项目里有用；主项目里想合并数组行为，得自己在写入前算好。

## 自查

1. 官方那句定义里给出了哪三个可以更新 app config 的时机？
2. 为什么 `app.config.ts` 里不能放密钥？官方 caution 原文是怎么说的？
3. `updateAppConfig` 的"深度赋值"带来哪两条后果？它能删掉一个键吗？
4. 在服务端路由里 `useAppConfig()` 的返回类型来自哪个接口？为什么和页面里不一样？
5. 直接给 `AppConfig` 写类型有什么副作用？官方给的注释是哪一句？
6. `app.config.ts` 与 `runtimeConfig` 的选边判据是什么？

## 下一篇

109 是视图部分的收束：从 `app.vue` 到 `<NuxtPage>`、布局、加载屏、进度条、播报，再到岛屿，把这一路串成一条完整的渲染链路。

## 本文依据

- `https://nuxt.com/docs/4.x/directory-structure/app/app-config`
  - 原文（描述）："Expose reactive configuration within your application with the App Config file."
  - 原文："Nuxt provides an `app/app.config.ts` config file to expose reactive configuration within your application with the ability to update it at runtime within lifecycle or using a nuxt plugin and editing it with HMR (hot-module-replacement)."
  - 原文："You can easily provide runtime app configuration using `app.config.ts` file. It can have either of `.ts`, `.js`, or `.mjs` extensions."
  - 原文（caution）："Do not put any secret values inside `app.config` file. It is exposed to the user client bundle."
  - 原文（note）："When configuring a custom `srcDir`, make sure to place the `app.config` file at the root of the new `srcDir` path."
  - 原文（Usage）："To expose config and environment variables to the rest of your app, you will need to define configuration in `app.config` file."；示例 `defineAppConfig({ theme: { primaryColor: '#ababab' } })`；"We can now universally access `theme` both when server-rendering the page and in the browser using `useAppConfig` composable."；页面侧 `const appConfig = useAppConfig()`、`console.log(appConfig.theme)`
  - 原文："The `updateAppConfig` utility can be used to update the `app.config` at runtime."
  - 原文（Typing App Config）："Nuxt tries to automatically generate a TypeScript interface from provided app config so you won't have to type it yourself."；"However, there are some cases where you might want to type it yourself."
  - 原文（App Config Input）："`AppConfigInput` might be used by module authors who are declaring what valid *input* options are when setting app config. This will not affect the type of `useAppConfig()`."；示例 `declare module 'nuxt/schema' { interface AppConfigInput { theme?: { primaryColor?: string } } }` + `export {}`
  - 原文（App Config Output）："If you want to type the result of calling `useAppConfig()` in app code, then you will want to extend `AppConfig`. Outside app code (server routes, the `shared/` directory and `nuxt.config`) `useAppConfig()` returns `SharedAppConfig`, so extend that interface instead."
  - 原文（warning）："Be careful when typing `AppConfig` as you will overwrite the types Nuxt infers from your actually defined app config."；示例注释 `// This will entirely replace the existing inferred \`theme\` property`、`// such as string literal types`、`primaryColor?: 'red' | 'blue'`、`// It is always important to ensure you import/export something when augmenting a type`
  - 原文（Merging Strategy）："Nuxt uses a custom merging strategy for the `AppConfig` within the layers of your application. This strategy is implemented using a Function Merger, which allows defining a custom merging strategy for every key in `app.config` that has an array as value."；（note）"The function merger can only be used in the extended layers and not the main `app.config` in project."
- `https://nuxt.com/docs/4.x/api/composables/use-app-config`
  - 原文："Access the reactive app config defined in the project."；示例 `const appConfig = useAppConfig()`、`console.log(appConfig)`
  - 原文 Source：`packages/nuxt/src/app/config.ts`
- `https://nuxt.com/docs/4.x/api/utils/update-app-config`
  - 原文："Update the App Config at runtime."
  - 原文（note）："Updates the `app.config` using deep assignment. Existing (nested) properties will be preserved."
  - 原文示例：`import { updateAppConfig, useAppConfig } from '#imports'`；`const appConfig = useAppConfig() // { foo: 'bar' }`；`const newAppConfig = { foo: 'baz' }`；`updateAppConfig(newAppConfig)`；`console.log(appConfig) // { foo: 'baz' }`
  - 原文 Source：`packages/nuxt/src/app/config.ts`（与 `useAppConfig` 同一份实现）
- `https://nuxt.com/docs/4.x/api/nuxt-config`
  - 对照用（见 072、073）：`runtimeConfig` —— "Runtime config allows passing dynamic config and environment variables to the Nuxt app context."；"The value of this object is accessible from server only using `useRuntimeConfig`. It mainly should hold *private* configuration which is not exposed on the frontend. This could include a reference to your API secret tokens. Anything under `public` and `app` will be exposed to the frontend as well. Values are automatically replaced by matching env variables at runtime, e.g. setting an environment variable `NUXT_API_KEY=my-api-key NUXT_PUBLIC_BASE_URL=/foo/` would overwrite the two values in the example below."
- `https://nuxt.com/docs/4.x/api/components/nuxt-page`
  - 对照用（见 091）：模板中读取响应式值随换页更新的上下文
