---
url: /posts/0ea7f64eb897f50a90786346e0b4fa08/
title: 用 Vite 插件接入 Nuxt：一条配置数组、addVitePlugin 与那条替代建议
date: 2027-05-27T09:00:00+08:00
updated: 2027-05-27T09:00:00+08:00
author: cmdragon

summary:
  摘要：这一页只解决一个问题：**当某个 Vite 插件正好满足需求时，怎么接进来而不必写模块**。项目侧的动作是两步——装包、把它放进 `nuxt.config.ts` 的 **`vite.plugins` 数组**；作者侧则要换成 **`addVitePlugin`**，并且**Nuxt 5+ 里要按环境生效的插件得用 `applyToEnvironment()` 方法**。页面用 `@rollup/plugin-yaml` 做例子，接完之后**就能直接 `import config from '~/data/hello.yaml'`**。最后那条 `::important` 是给进阶读者的方向指引：**要访问已解析的 Vite 配置，就应该写在 Vite 插件内部的 `config` 与 `configResolved` 钩子里，而不是用 Nuxt 的 `vite:extend`、`vite:extendConfig`、`vite:configResolved`**。

categories:
  - 前端开发

tags:
  - nuxt4
  - Vite插件
  - yaml导入
  - 配置数组
  - 插件时机
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

"我就想让 `.yaml` 能被 import"——这种事不值得写一个模块。这一页给的就是最短路径：两行配置。

## 这页的适用前提

开场句已经把取舍说了：**模块提供了大量功能，但有时候某个具体的 Vite 插件能更直接地满足你的需求**。

也就是说：**顺序是先看有没有现成 Vite 插件能直接解决，再考虑写模块**。这一页的存在就是把这条捷径写清楚。

## 第一步：装包

例子用的是 `@rollup/plugin-yaml`，安装给了五种包管理器：

```bash [npm]
npm install @rollup/plugin-yaml
```

```bash [yarn]
yarn add @rollup/plugin-yaml
```

```bash [pnpm]
pnpm add @rollup/plugin-yaml
```

```bash [bun]
bun add @rollup/plugin-yaml
```

```bash [deno]
deno add npm:@rollup/plugin-yaml
```

**只有 deno 那条带 `npm:` 前缀**（它需要从 npm 注册表解析），其余四条是裸包名。

## 第二步：放进 `vite.plugins`

```ts [nuxt.config.ts]
import yaml from '@rollup/plugin-yaml'

export default defineNuxtConfig({
  vite: {
    plugins: [
      yaml(),
    ],
  },
})
```

动作只有两个：**顶层 import 插件**、**在 `vite.plugins` 数组里放它的调用结果**（`yaml()` 带括号，是工厂返回值而不是函数本身）。

**这里没有用 `addVitePlugin`**——因为这是项目侧配置，`vite` 那一族配置项本来就直接透传给构建器。

## 第三步：直接用

配置完之后就可以在项目里放一个 YAML 文件并 import 它：

```yaml [data/hello.yaml]
greeting: "Hello, Nuxt with Vite!"
```

```vue [app/components/Hello.vue]
<script setup>
import config from '~/data/hello.yaml'
</script>

<template>
  <h1>{{ config.greeting }}</h1>
</template>
```

两处细节：**导入路径用的是 `~/data/hello.yaml`**（**`~/` 指向 `srcDir`，所以这个 `data/` 目录在 `app/` 下**）、**导入的是默认导出且直接拿到对象**（`config.greeting`），没有 `?raw`、没有 `?url` 这类后缀。

**这正是插件的作用**：让 `.yaml` 变成一个可 import 的模块。页面也这么收尾："现在我们已经安装并配置了 Vite 插件，就可以在项目里直接使用 YAML 文件"。

顺带一条值得注意的措辞：页面在配置与使用之间写的是 "**Now we installed and configured our Vite plugin, we can start using YAML files directly**"——**没有说明类型支持**（`.yaml` 导入的 TS 类型从哪来，这一页一个字没提）。要让 IDE 认这个导入，属于补类型声明那一类动作。

## 模块作者要换函数

页面的唯一小节标题是 **Using Vite Plugins in Nuxt Modules**，规则一句：**如果你在开发一个 Nuxt 模块并且需要添加 Vite 插件，应该使用 `addVitePlugin` 工具**：

```ts [modules/my-module.ts]
import { addVitePlugin, defineNuxtModule } from '@nuxt/kit'
import yaml from '@rollup/plugin-yaml'

export default defineNuxtModule({
  setup () {
    addVitePlugin(yaml())
  },
})
```

**同样是 `yaml()` 这个值，只是从配置数组换成了函数调用。**

紧接着是环境相关的那条：

```ts [modules/my-module.ts]
import { addVitePlugin, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    addVitePlugin(() => ({
      name: 'my-client-plugin',
      applyToEnvironment (environment) {
        return environment.name === 'client'
      },
      // Plugin configuration
    }))
  },
})
```

**"在 Nuxt 5+ 里要按环境区分的插件，用 `applyToEnvironment()` 方法"**，而这句里有两个信息值得单独看：**传给 `addVitePlugin` 的可以是函数**（这里返回一个插件对象），**并且判断依据是 `environment.name`**（示例用的值是 `'client'`）。

这与 Kit 里那条警告配套：**用 `server: false` 或 `client: false` 选项注册的插件，它的 `config` 与 `configResolved` 钩子在 Nuxt 5+ 不会被调用**——所以**别再用选项区分端，改用这个方法**。

## 那条 `::important` 指的路

页面最后一条重要提示是给进阶读者的方向修正：

> If you're writing code that needs to access resolved Vite configuration, you should use the `config` and `configResolved` hooks *within* your Vite plugin, rather than using Nuxt's `vite:extend`, `vite:extendConfig` and `vite:configResolved`.

它一次性排除了三个 Nuxt 钩子：**要读已解析的 Vite 配置，应该写在插件自己的 `config`／`configResolved` 钩子里，而不是挂 `vite:extend`、`vite:extendConfig`、`vite:configResolved`**。

对照钩子表，这条建议有额外依据：**`vite:extendConfig` 与 `vite:configResolved` 两行都标着 "Deprecated in Nuxt 5+"**（理由：Nuxt 5 作用于共享配置而不是分开的 client/server 配置）。也就是说**这三个钩子里被点名排除的两个正在退场**，而**官方示例页那段"把 Vite 配置取出来"用的正是 `vite:extend`**——**那条路能用，但同一条提示又把你往插件内部引**。

三条通道怎么选：

| 你要做的事 | 推荐做法 |
| --- | --- |
| 用现成插件加一种导入能力 | 项目侧进 `vite.plugins`；模块侧用 `addVitePlugin` |
| 只给某一端加的插件 | 插件对象里写 `applyToEnvironment` |
| 读或改已解析的 Vite 配置 | **在插件内部的 `config`／`configResolved` 钩子里做** |

页面在末尾放了一个跳转指向 Kit 的 `addVitePlugin` 条目，**没有讲 webpack 侧的对应写法**（webpack 的等价物在 Kit builder 那一页，`addWebpackPlugin`）。

这一页未记载的还有：**为什么例子选了一个 rollup 插件**（`@rollup/plugin-yaml` 名字里就是 rollup，页面没有解释它在 Vite 下的可用性依据）、`.yaml` 导入的类型声明怎么补、以及 `vite.plugins` 数组与 `addVitePlugin` 同时使用时插件的先后顺序。

下一篇把模板那套函数串成一次完整实践：一个模块选项怎么变成可 import 的文件。

## 本文依据

- https://nuxt.com/docs/4.x/guide/recipes/vite-plugin
