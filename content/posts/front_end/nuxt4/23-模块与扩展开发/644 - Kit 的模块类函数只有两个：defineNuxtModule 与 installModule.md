---
url: /posts/3c4668366eda2b86d5de98234c6db075/
title: Kit 的模块类函数只有两个：defineNuxtModule 与 installModule
date: 2027-05-14T11:00:00+08:00
updated: 2027-05-14T11:00:00+08:00
author: cmdragon

summary:
  摘要：Kit 里管"模块本身"的那页**只有两个函数**，而且**第二个已经废弃**：`defineNuxtModule` 与 `installModule`。`defineNuxtModule` 有两个重载——**直接传定义对象**那种返回 `NuxtModule`，**先写 `defineNuxtModule<TOptions>()` 再用 `.with({...})`** 那种是给合并后的选项拿准确类型用的。`installModule` 的调用面是 `moduleToInstall`（字符串或模块对象）、`inlineOptions`、`nuxt` 三个参数，废弃说明给的替代是 `moduleDependencies` 选项，并留了一句"未来会被移除（或可能变成非阻塞）"。这一页其余内容都是参数表与例子，**没有第三个函数**——注入资产的工具在别的页。

categories:
  - 前端开发

tags:
  - nuxt4
  - NuxtKit
  - defineNuxtModule
  - installModule
  - 模块定义
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

写模块时最容易高估的一件事是"Kit 里应该有个函数专门做这个"。**管模块定义的这一页其实只有两个函数，其中一个是留给旧代码的**。这一篇把这两个的形态与边界说清。

## 这一页的开场句

页面对模块的定位是一句：**模块是 Nuxt 的构件（building blocks）**。紧接着给了这一页存在的理由：**你可以用 `defineNuxtModule` 函数来定义一个模块，并用 `moduleDependencies` 选项来声明依赖**。

注意这句里两个动作分给了**同一个函数的两个键**——定义与依赖都不需要额外函数。这页只有 2 个条目的原因就在这里。

## 函数一：`defineNuxtModule`

定性句是：**定义一个 Nuxt 模块，自动合并默认值与用户提供的选项，安装提供的钩子，并调用一个可选的 setup 函数以获得完全控制**。

它有两个重载：

```ts
export function defineNuxtModule<TOptions extends ModuleOptions> (
  definition?: ModuleDefinition<TOptions, Partial<TOptions>, false> | NuxtModule<TOptions, Partial<TOptions>, false>,
): NuxtModule<TOptions, TOptions, false>

export function defineNuxtModule<TOptions extends ModuleOptions> (): {
  with: <TOptionsDefaults extends Partial<TOptions>> (
    definition: ModuleDefinition<TOptions, TOptionsDefaults, true> | NuxtModule<TOptions, TOptionsDefaults, true>,
  ) => NuxtModule<TOptions, TOptionsDefaults, true>
}
```

读这两个重载只需要抓住一件事：**第二个不接收参数，它返回一个带 `with` 方法的对象**。所以两种写法是：

```ts
// 形态一：直接给定义对象
export default defineNuxtModule({
  meta: { name: 'my-module', configKey: 'myModule' },
  defaults: { enabled: true },
  setup (options) { /* ... */ },
})

// 形态二：先给类型，再 with
export default defineNuxtModule<ModuleOptions>().with({
  meta: { name: '@nuxtjs/my-api', configKey: 'myApi' },
  defaults: { baseURL: 'https://api.example.com', timeout: 5000 },
  setup (resolvedOptions) { /* 这里的属性都有值 */ },
})
```

**什么时候必须用形态二**：当你希望 `setup` 收到的选项类型是"默认值补全之后的形状"。形态一里 TS 只看你写的接口，可选属性**即使有默认值也仍然标着可选**。合并规则、八个参数键、`configKey: false` 的禁用语义这些都在选项合并那一讲里，这里不重复。

## 函数二：`installModule`，已废弃

它上面挂的是一条 warning 类型的 callout：

> **Deprecated:** Use the `moduleDependencies` option in `defineNuxtModule` instead. The `installModule` function will be removed (or may become non-blocking) in a future version.

废弃之外还有一句现状描述：**以编程方式安装指定的 Nuxt 模块**，当你自己的模块依赖别的模块时有用；传给它的 `inlineOptions` 会被送到那个模块的 `setup` 函数。

签名与参数：

```ts
async function installModule (moduleToInstall: string | NuxtModule, inlineOptions?: any, nuxt?: Nuxt)
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `moduleToInstall` | `string \| NuxtModule` | `true` | 要安装的模块；**可以是模块名字符串，也可以是模块对象本身** |
| `inlineOptions` | `any` | `false` | 要传给该模块 `setup` 的选项对象 |
| `nuxt` | `Nuxt` | `false` | Nuxt 实例；不提供时**通过 `useNuxt()` 从上下文取** |

用法在 `setup` 里 `await`：

```ts
import { defineNuxtModule, installModule } from '@nuxt/kit'

export default defineNuxtModule({
  async setup () {
    // will install @nuxtjs/fontaine with Roboto font and Impact fallback
    await installModule('@nuxtjs/fontaine', {
      fonts: [
        {
          family: 'Roboto',
          fallbacks: ['Impact'],
          fallbackName: 'fallback-a',
        },
      ],
    })
  },
})
```

## 两者差在哪，为什么替代得动

| 维度 | `installModule` | `moduleDependencies` |
| --- | --- | --- |
| 写在哪 | `setup` 函数体内一次 `await` 调用 | 定义对象里的一个键 |
| 版本约束 | 无参数可写 | `version` 字段（semver） |
| 配置方向 | 只能传 inlineOptions | `overrides` 与 `defaults` 两个方向 |
| 缺失时 | 你自己判断分支 | `optional` 字段声明 |
| 能否按宿主配置动态决定 | 靠你在 `setup` 里写 if | 整个键**可以写成接收 nuxt 的函数** |

替代之所以成立，是因为右边那张表把"你在 `setup` 里手写的那段判断"变成了声明。**而 `moduleDependencies` 不能做的只有一件事：把一个已经拿到手的模块对象当依赖目标**——`installModule` 的 `moduleToInstall` 类型里包含 `NuxtModule`，声明式的键只接受字符串键。

**新代码没有理由再调它**。但这一页仍然把它写全了，因为它还在 Kit 的公开出口里；看旧模块源码时会遇到。

## 这一页没有的东西

三个常被误以为在这页的能力：

- **注入插件、组件、组合式函数**：分别在 plugins、components、autoimports 三页。
- **拿 `nuxt` 实例**：context 页的 `useNuxt` 与 `tryUseNuxt`（这一页的 `installModule` 只是**引用**了 `useNuxt()` 这个取法）。
- **路径解析**：`createResolver` 在 resolving 页，而几乎每个模块示例的前两行都用它。

也就是说这一页管的是"模块怎么被定义出来"，从定义往外的每一步都要换页查。

## 自查两行

1. 你的 `setup` 里有没有 `await installModule(...)`？**改成 `moduleDependencies` 后顺带把版本约束补上**（这是旧写法唯一给不了的东西）。
2. 你在 `setup` 里读选项时有没有写 `options.x ?? 默认值` 这类兜底？**有默认值却还需要兜底，通常是没用 `.with()`**，属于类型问题而不是运行时问题。

下一篇讲编程式用法那一页的五个函数——不用 `nuxt` 命令行动 Nuxt。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/modules
