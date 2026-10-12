---
url: /posts/501ae02abf4a68d635a5f8e3896f6cd6/
title: getLayerDirectories：十个目录字段、优先级顺序与那两条 note
date: 2027-05-20T11:00:00+08:00
updated: 2027-05-20T11:00:00+08:00
author: cmdragon

summary:
  摘要：图层这一页只有一个函数，但它把一件不好查的事标准化了：**`getLayerDirectories` 返回应用里每一层的十个已解析目录，并且数组顺序就是优先级顺序——第一个是用户/项目层（最高优先级），前面的层覆盖后面的，基础层排在最后**。页面明说它的价值在于**不必直接访问私有的 `nuxt.options._layers`**。返回值一张表十个字段（`root`、`server`、`modules`、`shared`、`app`、`public`、`appLayouts`、`appMiddleware`、`appPages`、`appPlugins`），**每个的默认形态在接口注释里给出**（例如 `app` 是 Nuxt 的 `srcDir`，默认 `/app/`；`appPages` 默认 `/app/pages`）。两条 `::note` 是常被忽略的细节：**结果用 WeakMap 缓存，反复调用不会重复计算**；**返回的路径一律带结尾斜杠**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 图层
  - 目录解析
  - 优先级
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块一旦要处理"每一层都有同名目录"这件事，就需要一份按优先级排好的目录清单。这一页给的就是那份清单，而且它把私有属性藏了起来。

## 为什么要有这个函数

开场句：**Nuxt 图层提供了一种跨项目共享与扩展功能的强大方式；在模块里处理图层时，你常常需要访问每一层的目录路径**。它要解决的正是这个：**`getLayerDirectories` 让你访问应用里所有图层已解析的目录路径**。

函数自己的定性把动机说得更直接：

> Get the resolved directory paths for all layers in a Nuxt application. This function provides a structured way to access layer directories without directly accessing the private `nuxt.options._layers` property.

**关键词是 private**：`nuxt.options._layers` 是私有属性，**这个函数是它受支持的替代入口**。在模块里看到直接读 `_layers` 的代码，应该换掉。

```ts
function getLayerDirectories (nuxt?: Nuxt): LayerDirectories[]
```

参数只有一个、可选：**要从中取图层的 Nuxt 实例；不提供时使用当前的 Nuxt 上下文**（与上下文那一页的取法一致）。

## 十个字段与它们的默认形态

返回数组里每一项是一个 `LayerDirectories`，**Return Value 一节给的是十字段表**：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `root` | `string` | 该层的根目录（**等价于 `rootDir`**） |
| `server` | `string` | Nitro 服务端代码所在目录 |
| `modules` | `string` | 本地模块目录 |
| `shared` | `string` | 客户端与服务端共用的代码目录 |
| `app` | `string` | 该层的源码目录（**等价于 `srcDir`**） |
| `public` | `string` | 静态资源的 public 目录 |
| `appLayouts` | `string` | Vue 布局组件目录 |
| `appMiddleware` | `string` | 路由中间件目录 |
| `appPages` | `string` | 文件式路由的页面目录 |
| `appPlugins` | `string` | Nuxt 插件目录 |

默认值只写在接口定义的注释里，**这一组注释给出的形态很关键**：

```ts
interface LayerDirectories {
  /** Nuxt rootDir (`/` by default) */
  readonly root: string
  /** Nitro source directory (`/server` by default) */
  readonly server: string
  /** Local modules directory (`/modules` by default) */
  readonly modules: string
  /** Shared directory (`/shared` by default) */
  readonly shared: string
  /** Public directory (`/public` by default) */
  readonly public: string
  /** Nuxt srcDir (`/app/` by default) */
  readonly app: string
  /** Layouts directory (`/app/layouts` by default) */
  readonly appLayouts: string
  /** Middleware directory (`/app/middleware` by default) */
  readonly appMiddleware: string
  /** Pages directory (`/app/pages` by default) */
  readonly appPages: string
  /** Plugins directory (`/app/plugins` by default) */
  readonly appPlugins: string
}
```

读这份注释能确认两件事：**五个字段全部是 `readonly`**（**只读，改了不会生效**）、**app 那一族的默认前缀就是 `app/`**，与 4.x 新目录结构一致（`appPages` 默认 `/app/pages`）。

页面在字段表之外还有一行总结：**返回的对象是"数组中每一层各一个"**，即**一个图层一个对象**。

## 顺序就是优先级

这是这一页最该记住的一段。原文：**图层按优先级排序，其中**：

- **第一个图层是用户/项目图层**（最高优先级）
- **数组里靠前的图层覆盖靠后的图层**
- **基础图层（base layers）出现在数组最后**（最低优先级）

并补了一句理由：**这个顺序与 Nuxt 的图层解析系统一致，用户定义的配置与文件优先于基础图层**。

所以任何"同名取哪一个"的判断，**都是从数组头往尾走、先命中的赢**，不需要自己排优先级。

## 两条 note：缓存与结尾斜杠

第一条：**这个函数用一个 WeakMap 做缓存，避免为同样的图层反复重新计算目录路径，从而在多次调用时提升性能**。

第二条：**该函数返回的目录路径总是带一个结尾斜杠，以保持一致**。

第二条有实操影响：**拼接时不要再自己加 `/`**（`layer.app + '/pages'` 会得到双斜杠），而**与 `join`/`resolve` 混用是安全的**——下面的官方示例正是这么写的。

## 四个示例覆盖四种用法

**遍历每一层**（这是入门形态，注意 `layer.appPages` 这类字段直接可用）：

```ts
import { defineNuxtModule, getLayerDirectories } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const layerDirs = getLayerDirectories()

    // Access directories from all layers
    for (const [index, layer] of layerDirs.entries()) {
      console.log(`Layer ${index}:`)
      console.log(`  Root: ${layer.root}`)
      console.log(`  App: ${layer.app}`)
      console.log(`  Server: ${layer.server}`)
      console.log(`  Pages: ${layer.appPages}`)
      // ... other directories
    }
  },
})
```

**跨所有层收集文件**（组件扫描，注释里再次强调顺序）：

```ts
import { defineNuxtModule, getLayerDirectories } from '@nuxt/kit'
import { resolve } from 'pathe'
import { globby } from 'globby'

export default defineNuxtModule({
  async setup () {
    const layerDirs = getLayerDirectories()

    // Find all component files across layers
    // Note: layerDirs[0] is the user layer (highest priority)
    // Later layers in the array have lower priority
    const componentFiles = []
    for (const [index, layer] of layerDirs.entries()) {
      const files = await globby('**/*.vue', {
        cwd: resolve(layer.app, 'components'),
        absolute: true,
      })
      console.log(`Layer ${index} (${index === 0 ? 'user' : 'base'}):`, files.length, 'components')
      componentFiles.push(...files)
    }
  },
})
```

`cwd` 那里用 `resolve(layer.app, 'components')`——**这就是"路径自带结尾斜杠"仍要与 resolve 一起用的原因**。**示例的 `index === 0 ? 'user' : 'base'` 也是优先级写法的最短表达**。

**给每一层各注册一份模板**（用层根目录名做区分）：

```ts
import { addTemplate, defineNuxtModule, getLayerDirectories } from '@nuxt/kit'
import { basename, resolve } from 'pathe'
import { existsSync } from 'node:fs'

export default defineNuxtModule({
  setup () {
    const layerDirs = getLayerDirectories()

    // Add a config file from each layer that has one
    for (const dirs of layerDirs) {
      const configPath = resolve(dirs.app, 'my-module.config.ts')
      if (existsSync(configPath)) {
        addTemplate({
          filename: `my-module-${basename(dirs.root)}.config.ts`,
          src: configPath,
        })
      }
    }
  },
})
```

**`basename(dirs.root)` 把层名放进模板文件名**，避免不同层的同名配置互相覆盖——这是"每层都能贡献一份"的标准手法。

**尊重优先级取第一份**，以及它的反向合并变体：

```ts
    let configContent = null
    for (const dirs of layerDirs) {
      const configPath = resolve(dirs.app, 'my-config.json')
      if (existsSync(configPath)) {
        configContent = readFileSync(configPath, 'utf-8')
        console.log(`Using config from layer: ${dirs.root}`)
        break // Use the first (highest priority) config found
      }
    }

    // Alternative: Collect configs from all layers, with user layer taking precedence
    const allConfigs = {}
    for (const dirs of layerDirs.reverse()) { // Process from lowest to highest priority
      const configPath = resolve(dirs.app, 'my-config.json')
      if (existsSync(configPath)) {
        const config = JSON.parse(readFileSync(configPath, 'utf-8'))
        Object.assign(allConfigs, config) // Later assignments override earlier ones
      }
    }
```

两种策略要分清：**要"整份来自最高优先级的那一层"就从头找、找到就 `break`**；**要"逐键合并、用户的键最后生效"就 `reverse()` 从最低优先级开始，让后面的赋值覆盖前面的**。

最后一个是筛选有某目录的层（这个模式常用来统计"哪些层带了 assets"）：

```ts
    const layersWithAssets = layerDirs.filter((layer) => {
      return existsSync(resolve(layer.app, 'assets'))
    })
```

**注意它查的是 `layer.app` 下的 `assets`，而返回字段里并没有 `appAssets` 这一项**——**十个字段之外的目录要自己用 `resolve` 拼**。

## 三个使用边界

- **它是只读清单**：字段全是 `readonly`，**改这里的值不会改到构建行为**；要改目录应回到配置（`srcDir`、`serverDir`、`dir.*` 那族键）。
- **它不合并文件**：函数只给路径，"同名谁赢"由你的循环策略决定（上一节那两种）。
- **它不告诉你哪层是哪个模块**：返回值里没有图层的名字或来源，只有 `root`；**要识别层得靠 `basename(dirs.root)` 这类自己拼的办法**。

这一页没写的还有：`getLayerDirectories` 与 `extends` 的顺序是否严格一致、基础层没有 `app/` 目录时对应字段返回什么（路径仍然给出还是缺失）、以及 WeakMap 的键是什么。

下一篇离开 Kit，进入模块开发流程的另一些环节。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/layers
