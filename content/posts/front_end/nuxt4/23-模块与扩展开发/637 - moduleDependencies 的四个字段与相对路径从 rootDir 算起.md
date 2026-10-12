---
url: /posts/f594f8695e7e80f9a3192733896ad6b0/
title: moduleDependencies 的四个字段与相对路径从 rootDir 算起
date: 2027-05-12T10:00:00+08:00
updated: 2027-05-12T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：模块要依赖别的模块，**不再调用 `installModule`**，而是在定义里写 `moduleDependencies`。这一页说清了三件被代理的事：**按正确顺序安装**、**校验你给的版本约束**、**合并你为它们提供的配置**。四个字段各有各的边界：`version` 是 semver 范围，不满足就**启动直接报错**，但它**只在依赖能解析到 `package.json` 时生效，对项目本地模块等于空操作**；`overrides` 压在 `nuxt.options` 之上、`defaults` 垫在 `nuxt.options` 之下；`optional: true` 表示缺失时不自动装，但**只要那个模块在别处装了，你给的 overrides 与 defaults 照样生效**。最狠的一条是路径：相对路径**从项目 `rootDir` 算起，不是从声明它的那个文件算起**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块依赖
  - 版本约束
  - 配置合并
  - 分层
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一个模块要用 Tailwind，你有两种做法：让每个用户自己去装、自己去配；或者在模块里声明依赖。**后者的坑集中在两处：路径怎么解析、版本约束什么时候不生效**。这一页讲的就是这两处。

## 声明依赖换来什么

原话给的收益是三件：**Nuxt 会确保这些模块按正确的顺序安装、校验你提供的任何版本约束、并合并你为它们提供的配置**。

一个完整例子（依赖 Tailwind 并把模块自己的运行时文件喂进它的 content 配置）：

```ts
import { createResolver, defineNuxtModule } from '@nuxt/kit'

const resolver = createResolver(import.meta.url)

export default defineNuxtModule<ModuleOptions>({
  meta: {
    name: 'my-module',
  },
  moduleDependencies: {
    '@nuxtjs/tailwindcss': {
      // You can specify a version constraint for the module
      version: '>=6',
      // Any configuration that should override `nuxt.options`
      overrides: {
        exposeConfig: true,
      },
      // Any configuration that should be set. It will override module defaults but
      // will not override any configuration set in `nuxt.options`
      defaults: {
        config: {
          darkMode: 'class',
          content: {
            files: [
              resolver.resolve('./runtime/components/**/*.{vue,mjs,ts}'),
              resolver.resolve('./runtime/*.{mjs,js,ts}'),
            ],
          },
        },
      },
    },
  },
  setup (options, nuxt) {
    // We can inject our CSS file which includes Tailwind's directives
    nuxt.options.css.push(resolver.resolve('./runtime/assets/styles.css'))
  },
})
```

这一页有一条 `::callout` 直接做了新旧交接：**`moduleDependencies` 选项取代了被废弃的 `installModule` 函数**。

## 键是什么

**每个条目的键标识要依赖哪个模块**，可以取三种值：

| 键的形态 | 例子 | 适用 |
| --- | --- | --- |
| npm 包名 | `'@nuxtjs/tailwindcss'` | 依赖装在你包依赖里的官方或社区模块 |
| 本地模块目录的路径 | `'./modules/my-local-module'` | 依赖和被依赖的模块都在同一个项目里 |
| Nuxt 别名 | `'~/modules/another-local-module'` | 同上，但不想纠结相对层级 |

## 相对路径那条警告值得单独抄一遍

依赖住在项目的 `modules/` 目录里时，用文件路径声明：

```ts
export default defineNuxtModule({
  moduleDependencies: {
    // Path relative to the project root
    './modules/my-local-module': {},
    // Or using a Nuxt alias
    '~/modules/another-local-module': {},
  },
  // ...
})
```

`::important` 把最容易写错的那一点挑明了：

> **Relative paths are resolved from your project's `rootDir`**, not from the file declaring the dependency. A module at `modules/foo.ts` referencing `modules/bar.ts` must use `'./modules/bar'`, not `'./bar'`.

也就是：**`modules/foo.ts` 里想引用同目录的 `bar.ts`，要写 `'./modules/bar'`**。写成 `'./bar'` 会被当成从项目根算起，于是指向一个不存在的 `bar`。页面紧接着给的规避方法是**用别名 `~/modules/bar`**，它把歧义消掉了——这也是这一页自己的第二个示例采用的写法。

顺带对照一下：`overrides`/`defaults` 里的路径走的是 `resolver.resolve(...)`，那是**以当前模块文件为基准**的解析器。**同一个对象里两种基准并存**，这是最容易读漏的地方。

## 四个字段的边界

| 字段 | 文档原文要点 | 容易被忽略的限定 |
| --- | --- | --- |
| `version` | 一个 semver 范围；解析到的模块版本不满足时 **Nuxt 抛错** | **只在依赖能解析到 `package.json` 时适用，对项目本地模块是空操作**（no-op） |
| `overrides` | 应用**在 `nuxt.options` 之上**的配置，优先于用户配置 | 它是"压过用户"的那一档，用错会让用户改不动 |
| `defaults` | 应用**在 `nuxt.options` 之下**的配置，用户配置优先于它 | 注释补了一句：它会盖过模块自身的默认值，但**不盖过用户写过的值** |
| `optional` | 为 `true` 时**缺失不会被自动安装**；如果模块在别处装了，`overrides` 与 `defaults` **仍然应用** | 它改变的是"装不装"，不改变"配不配" |

`version` 那条限定最实用：**你依赖项目里的本地模块时写版本约束不会有任何效果**，不报错、也不检查，静默跳过。想约束版本，只能约束能解析到 `package.json` 的那些依赖。

`optional` 的另一面也值得记：**它是"可选依赖不会被安装，但保证它装了之后选项能被设置"**——Kit 页那个 fontaine 示例就是这么用的：

```ts
'@nuxtjs/fontaine': {
  // Optional dependencies won't be installed but ensure that options
  // can be set if they _are_ installed
  optional: true,
  defaults: {
    fonts: [
      {
        family: 'Roboto',
        fallbacks: ['Impact'],
      },
    ],
  },
},
```

## 合并的方向

把 `overrides` 与 `defaults` 的措辞并排放，就得到这条链（从高到低）：

| 优先级 | 来源 |
| --- | --- |
| 最高 | 你写的 `overrides` |
| 其次 | 用户在 `nuxt.config` 里写的配置（即 `nuxt.options`） |
| 再次 | 你写的 `defaults` |
| 最低 | 被依赖模块自己的默认值 |

**这就是"什么时候用 defaults、什么时候用 overrides"的全部判据**：想让用户的配置仍然赢，用 `defaults`；想强制一个值，才用 `overrides`。

## 还能写成函数

依赖不必是静态对象。**`moduleDependencies` 可以是一个接收 Nuxt 实例的函数**，返回同样的记录，于是能按宿主配置条件化：

```ts
moduleDependencies (nuxt) {
  const dependencies: Record<string, any> = {
    '@nuxtjs/tailwindcss': {
      version: '>=6.0.0',
    },
  }

  // Conditionally add dependencies based on Nuxt config
  if (nuxt.options.experimental?.someFeature) {
    dependencies['@nuxtjs/fontaine'] = {
      optional: true,
    }
  }

  return dependencies
}
```

配套的说明在 Kit 页那句注释里：**你的 `setup` 逻辑在所有依赖初始化之后才跑**。

## 一处口径差别，别当成保证

这一页开头说 Nuxt 会**确保这些模块按正确的顺序安装**，而模块结构页里 `moduleDependencies` 的注释写的是另一回事：

> Configuration for other modules - this does not ensure the module runs before your module, but it allows you to change the other module's configuration before it runs

两句并不完全同义：**前者说的是依赖被安装这件事的顺序，后者明确说它不保证那个模块跑在你的模块之前，只是让你在它运行前改它的配置**。所以"靠依赖声明来编排两个模块的执行先后"不能当成文档给过的保证；真要控制时序，得回到生命周期钩子那一侧（`modules:done`）。

下一篇讲发布前的打包：`nuxt build-module` 那一个参数和六个选项。

## 本文依据

- https://nuxt.com/docs/4.x/guide/modules/module-dependencies
- https://nuxt.com/docs/4.x/api/kit/modules
