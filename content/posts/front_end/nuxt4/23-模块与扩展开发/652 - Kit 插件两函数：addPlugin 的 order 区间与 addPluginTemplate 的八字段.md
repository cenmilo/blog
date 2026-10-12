---
url: /posts/fce9fcc83bed0dcbefca1abf95242c1d/
title: Kit 插件两函数：addPlugin 的 order 区间与 addPluginTemplate 的八字段
date: 2027-05-17T10:00:00+08:00
updated: 2027-05-17T10:00:00+08:00
author: cmdragon

summary:
  摘要：模块带插件有两个函数：**`addPlugin` 注册一个已有的插件文件，`addPluginTemplate` 先生成代码再把它注册成插件**（后者用于**需要在构建期生成代码**的情况，Nuxt 内部就用它来生成 Vue app 配置）。关于执行顺序，这一页给了整套数字：**数字小的先跑，用户插件默认 `0`；官方建议把 `order` 取在 `-20`（跑在 Nuxt 插件之前）到 `20`（跑在之后）之间**——同时两条 `::warning` 又说**非必要不要用 `order`，只是要排在 Nuxt 默认之后就用 `append`**，而 `append` **默认 `false`，也就是默认插到数组前面**。`mode` 有三个值（`all`/`server`/`client`），**也可以在 `src` 上用 `.client`、`.server` 后缀**达到同样效果。

categories:
  - 前端开发

tags:
  - nuxt4
  - 插件注册
  - 插件顺序
  - NuxtKit
  - 代码生成
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

插件的顺序问题比"能不能注入"更难查：跑早了拿不到别人提供的东西，跑晚了又来不及改配置。这一页把顺序的数字规则一次给全了。

## 为什么模块要显式注册

开场两句：**插件是自包含的代码，通常给 Vue 加上应用级的功能；在 Nuxt 里，插件会从 `app/plugins/` 目录自动导入**。**如果你需要随模块一起发布插件，Nuxt Kit 提供 `addPlugin` 与 `addPluginTemplate` 两个方法**。

这句里的对照很重要：**自动扫描只覆盖项目自己的 `app/plugins/`**，模块 runtime 里的插件文件**不会因为存在就被加载**，必须注册。

## `addPlugin`：注册一个已有文件

定性：**注册一个 Nuxt 插件并加入 plugins 数组**。

```ts
function addPlugin (plugin: NuxtPlugin | string, options?: AddPluginOptions): NuxtPlugin
```

三个信息藏在签名里：**第一个参数可以是插件对象或字符串**（给字符串时会被转成插件对象，`src` 设为那个字符串值）、**第二个参数可选**、**它有返回值 `NuxtPlugin`**。

插件对象三个字段：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `src` | `string` | `true` | 插件文件路径 |
| `mode` | `'all' \| 'server' \| 'client'` | `false` | `'all'` 时插件同时进客户端与服务端两个 bundle；`'server'` 只进服务端；`'client'` 只进客户端。**另外可以在指定 `src` 时用 `.client` 与 `.server` 后缀**来只在一端使用 |
| `order` | `number` | `false` | 插件顺序；**数字小的先跑，用户插件默认是 `0`**；建议取 **`-20` 到 `20`** 之间——`-20` 对应 pre 插件（跑在 Nuxt 插件之前），`20` 对应 post 插件（跑在 Nuxt 插件之后） |

`order` 那格的原文还有一句限定：**这允许更细粒度地控制插件顺序，只应由高级用户使用**。

第二个参数只有一项：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `append` | `boolean` | `false` | 为 `true` 时插件被追加到 plugins 数组；**为 `false` 时插到前面。默认 `false`** |

两条 `append` 与 `order` 的关系由一条 `::warning` 直接给出：

> Avoid using `order` unless necessary. Use `append` if you simply need to register plugins after Nuxt defaults.

也就是**"排在 Nuxt 默认之后"这种需求应该用 `append`，而不是写一个 `order: 20`**。

示例是一对文件：

```ts [module.ts]
import { addPlugin, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const { resolve } = createResolver(import.meta.url)

    addPlugin({
      src: resolve('runtime/plugin.js'),
      mode: 'client',
    })
  },
})
```

```ts [runtime/plugin.ts]
export default defineNuxtPlugin((nuxtApp) => {
  const colorMode = useColorMode()

  nuxtApp.hook('app:mounted', () => {
    if (colorMode.preference !== 'dark') {
      colorMode.preference = 'dark'
    }
  })
})
```

这个 runtime 示例里 **`useColorMode()` 没有 import**，而模块结构那一页明确警告过：**发布出去的模块在 `node_modules` 里不启用自动导入，必须从 `#imports` 之类显式导入**。两处口径不一致——**照抄示例发到 npm 会出问题，实际写插件时要自己补上显式导入**。

另外注意 `mode: 'client'` 与那个只在 `app:mounted` 做的事是配套的：**客户端才有的动作，就别让它进服务端 bundle**。

## `addPluginTemplate`：先造代码再当插件

定性：**添加一个模板并把它注册成 Nuxt 插件。这适用于需要在构建期生成代码的插件**。

```ts
function addPluginTemplate (pluginOptions: NuxtPluginTemplate, options?: AddPluginOptions): NuxtPlugin
```

与 `addPlugin` 的一处签名差别值得注意：**第一个参数的类型里没有 `string`**——**只能给模板对象，不能给路径字符串**。

模板对象八个字段：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `src` | `string` | `false` | 模板路径；**不给 `src` 就必须给 `getContents`** |
| `filename` | `string` | `false` | 模板文件名；不给时从 `src` 生成，**这种情况下 `src` 必需** |
| `dst` | `string` | `false` | 目标路径；不给时由 `filename` 与 Nuxt 的 `buildDir` 生成 |
| `mode` | `'all' \| 'server' \| 'client'` | `false` | 与 `addPlugin` 那一格**逐字相同**（含 `.client`/`.server` 后缀那句） |
| `options` | `Record<string, any>` | `false` | 传给模板的选项 |
| `getContents` | `(data: Record<string, any>) => string \| Promise<string>` | `false` | 以 `options` 对象被调用，返回字符串或 promise；**给了 `src` 时这个函数会被忽略** |
| `write` | `boolean` | `false` | **为 `true` 时写入目标文件，否则只存在于虚拟文件系统** |
| `order` | `number` | `false` | 与 `addPlugin` 那一格**逐字相同**（默认 0、建议 -20 到 20） |

第二条 `::warning` 在这里变成了两句：

> Prefer using `getContents` for dynamic plugin generation. Avoid setting `order` unless necessary.

**动态生成插件代码优先用 `getContents`**。基本用法就是把内容拼成字符串：

```ts
import { addPluginTemplate, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options) {
    addPluginTemplate({
      filename: 'module-plugin.mjs',
      getContents: () => `import { defineNuxtPlugin } from '#app/nuxt'
export default defineNuxtPlugin({
  name: 'module-plugin',
  setup (nuxtApp) {
    ${options.log ? 'console.log("Plugin install")' : ''}
  }
})`,
    })
  },
})
```

三处细节：**模板里第一行自己 `import { defineNuxtPlugin } from '#app/nuxt'`**（生成的是普通文件，没有自动导入帮你）、`filename` 用 `.mjs`、**用三元把一行代码按需插进模板**。

## Nuxt 自己怎么用这个函数

页面给的第二组例子最有信息量，因为它说明**这不是给第三方用的边缘能力**：

> For example, Nuxt internally uses this function to generate Vue app configurations.

框架内部就用它来生成 Vue 应用配置。完整代码：

```ts
import { addPluginTemplate, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup (_, nuxt) {
    if (nuxt.options.vue.config && Object.values(nuxt.options.vue.config).some(v => v !== null && v !== undefined)) {
      addPluginTemplate({
        filename: 'vue-app-config.mjs',
        write: true,
        getContents: () => `import { defineNuxtPlugin } from '#app/nuxt'
export default defineNuxtPlugin({
  name: 'nuxt:vue-app-config',
  enforce: 'pre',
  setup (nuxtApp) {
    ${Object.keys(nuxt.options.vue.config!)
        .map(k => `nuxtApp.vueApp.config[${JSON.stringify(k)}] = ${JSON.stringify(nuxt.options.vue.config![k as 'idPrefix'])}`)
        .join('\n')
    }
  }
})`,
      })
    }
  },
})
```

它把"配置对象里有什么"翻译成"一行行赋值语句"，输入输出对照是这样：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  vue: {
    config: {
      idPrefix: 'nuxt',
    },
  },
})
```

```ts [#build/vue-app-config.mjs]
import { defineNuxtPlugin } from '#app/nuxt'

export default defineNuxtPlugin({
  name: 'nuxt:vue-app-config',
  enforce: 'pre',
  setup (nuxtApp) {
    nuxtApp.vueApp.config.idPrefix = 'nuxt'
  },
})
```

这段示范了四个做法：**先判断有没有非空值再决定要不要注册插件**（`Object.values(...).some(...)`）、**`write: true` 让生成的文件落盘**、**`name` 用 `nuxt:` 前缀**、**`enforce: 'pre'` 让插件本体声明自己排在前面**。最后一项和 `order` 是两套机制，页面**没有说明它们同时存在时的关系**。

## 顺序这件事的完整图

把这一页散落的四处合起来：

| 机制 | 出处 | 效果 |
| --- | --- | --- |
| `order` 数字 | 两个插件对象的表 | **越小越先跑；用户插件默认 `0`；`-20` 是 pre、`20` 是 post** |
| `append` | 两个函数的 `options` | **默认 `false`＝插到数组前面**；`true` 才追加 |
| `enforce: 'pre'` | 生成代码里的插件本体属性 | 框架内部示例用它（含义在本页未解释） |
| 警告本身 | 两条 `::warning` | **非必要不要用 `order`**；要排在默认之后就用 `append` |

结论是：**先考虑 `append`，再考虑 `order`，`enforce` 属于生成代码时的另一种写法**。

这一页没记载的：`addPlugin` 返回的那个 `NuxtPlugin` 能拿来做什么、`append` 与 `order` 同时给出时的优先级、以及 Nuxt 自带插件分别占在哪个 `order` 上（只知道用户默认 0、边界建议 ±20）。

下一篇继续 Kit 的其余分类。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/plugins
