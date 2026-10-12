---
url: /posts/6871b64ec97e73b3f64873ebf1897471/
title: builder：换成 webpack、rspack 或自己的实现
date: 2027-04-15T11:00:00+08:00
updated: 2027-04-15T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`builder` 管的是**"打包 Vue 那一侧应用"用哪个构建器**——官方定义句是 "The builder to use for bundling the Vue part of your application."，类型 `'vite' | 'webpack' | 'rspack' | string | { bundle: (nuxt: Nuxt) => Promise<void> }`，**默认 `"@nuxt/vite-builder"`**。官方说明同时给出三条路径：写简写名、写**导出 `bundle` 函数的包名**、或直接传一个带 `bundle` 的对象。两条限制要提前知道：**用 `webpack` 或 `rspack` 时必须显式安装 `@nuxt/webpack-builder`／`@nuxt/rspack-builder`**；而**只有那三个官方构建器标识符会被 Nuxt 按名字识别**（自定义的要靠通用契约工作）。另有一处容易混：**`builder` 与 `server.builder` 是两个键**，后者管服务端构建器（默认 `@nuxt/nitro-server`），而**可部署的服务端是 Nitro 产出的，不是 builder 产出的**。

categories:
  - 前端开发

tags:
  - nuxt4
  - builder
  - vite
  - webpack
  - 构建
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想换构建器，第一反应是"改一行 `builder` 就完了"。真动手时会先撞到一个包没装，然后再发现服务端那侧还有一个同名选项。

## 定义与三条写法

`## builder` 的两句：

> The builder to use for bundling the Vue part of your application.
>
> Nuxt supports multiple builders for the client-side application. By default, Vite is used, but you can switch to webpack, Rspack, or even provide a custom builder implementation.

- **类型**：`'vite' | 'webpack' | 'rspack' | string | { bundle: (nuxt: Nuxt) => Promise<void> }`
- **默认**：`"@nuxt/vite-builder"`

官方把三种用法都写成了代码。**写法一，简写名**（示例里同时注释掉前两个，选第三个）：

```ts
export default defineNuxtConfig({
  // default - uses @nuxt/vite-builder
  // builder: 'vite',

  // uses @nuxt/webpack-builder
  // builder: 'webpack',

  // uses @nuxt/rspack-builder
  builder: 'rspack',
})
```

注意一个细节：**默认值是 `"@nuxt/vite-builder"`（包名形态），而简写 `'vite'` 注释成"default"**——两种写法指向同一个构建器。

**写法二，自定义对象**（带 `bundle` 函数）：

```ts
export default defineNuxtConfig({
  builder: {
    async bundle (nuxt) {
      const entry = await resolvePath(resolve(nuxt.options.appDir, 'entry'))

      // Build client and server bundles
      await buildClient(nuxt, entry)
      if (nuxt.options.ssr) {
        await buildServer(nuxt, entry)
      }

      // ... it's a bit more complicated than that, of course!
    },
  },
})
```

这段里那句括号注释很实在：**"it's a bit more complicated than that, of course!"** ——示例只演了骨架：拿到入口、构建客户端、**只有 `nuxt.options.ssr` 为真时才构建服务端**。

**写法三，独立包**：**"To create a custom builder as a separate package, it should export a `bundle` function. You can then specify the package name in your `nuxt.config.ts`"**，示例是 `builder: 'my-custom-builder'`。

## 两条限制

**第一条：换构建器要先装包。** 官方那句是条件句：**"If you are using `webpack` or `rspack` you will need to make sure `@nuxt/webpack-builder` or `@nuxt/rspack-builder` is explicitly installed in your project."**

所以换 `'webpack'` 之后报"找不到构建器"是正常的——**这一节没给安装命令**，按你自己项目的包管理器装对应包即可（`webpack` 那节还提到要配置加载器与插件的选项，都在 `webpack` 键下）。

**第二条：按名字识别的只有三个。** builders 那页写得更硬：**"Only the three official builder specifiers (`@nuxt/vite-builder`, `@nuxt/webpack-builder`, `@nuxt/rspack-builder`) are recognised by name for builder-specific behaviour elsewhere in Nuxt. A custom builder still works through the generic contract described below."**

这条的实际含义是：**"构建器专属行为"只对这三个包名生效**——`webpack` 那一整节、`vite` 那一整节的选项，都是构建器专属的；自定义构建器读不到它们，只能走通用契约。

## 别忘了还有 server.builder

`builder` 与 **`server.builder`** 是两个键。后者在配置参考里的说明是 "Configuration for Nuxt's server builder."，其子项 `builder` 的默认是 `"@nuxt/nitro-server"`，并且带一条警告：**这是给内部使用、API 尚未定型的选项**。

两句 builders 页的话把分工讲清了：

> The deployable server itself is produced by Nitro via `@nuxt/nitro-server`, not by the builder. The builder hands its outputs to Nitro through a typed contract; Nitro bundles them into the final `.output`.

也就是说：**`builder` 换的是客户端（以及 SSR 应用入口）的打包方式，`.output` 里那个能部署的服务端始终出自 Nitro。** 换 builder 不等于换部署形态。

同页另一句也是这个意思的正面表述：**"Nuxt separates what to build from how to build it."**——Nuxt 核心与模块生成虚拟应用（入口、路由表、插件、组件注册表等在 `#build` 下的东西），**builder 负责把它变成真正的 JS 与 CSS bundle**，开发模式下还要起 dev server 并热更新。

## 一次换构建器的最小验证顺序

按文档能给的做法：

1. 装对应包（`@nuxt/webpack-builder` 或 `@nuxt/rspack-builder`）。
2. 把 `builder` 设成简写名或包名。
3. 跑 `npx nuxt build`，**看产物是否仍生成 `.output`、`.output/server/index.mjs` 是否存在**（Nitro 那一步不受影响）。
4. 再检查依赖构建器的配置是否要搬家：换到 webpack 后，原来写在 `vite` 下的选项不再起作用，对应位置是 `webpack`（例如资源引用处理、`loaders.vue` 之类）。

**第 4 步是本系列给的排查顺序，官方只说明"配置写在对应键下"这一层，没有给迁移清单。**

## 自查

- `builder` 只管"打包 Vue 那一侧"，默认 `"@nuxt/vite-builder"`；三种写法是简写名、导出 `bundle` 的包名、带 `bundle` 的对象。
- 换 `webpack`／`rspack` **必须显式安装对应包**，本节不给安装命令。
- **只有三个官方标识符会被按名字识别**用于"构建器专属行为"；自定义构建器走通用契约，读不到 `vite`／`webpack` 那些键。
- `builder` 与 `server.builder` 是两个键；可部署的服务端由 Nitro（`@nuxt/nitro-server`）产出，builder 只把输出通过契约交给它。
- Nuxt 核心与模块负责生成虚拟应用，builder 负责变成真实 bundle 并在开发模式起 dev server（"separates what to build from how to build it"）。

## 下一篇

下一篇 558 读 `build` 节那三项：`analyze`（默认三项 JSON 与 `template: treemap`、报告落在 `.nuxt/analyze/{name}.html`）、`templates`（官方直接建议改用 Kit 的 `addTemplate`）、`transpile`（四种元素形态，而那句"函数会收到一个对象"的括号在文档里是空的）。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## builder` 全节：两句定义、类型联合与默认 `"@nuxt/vite-builder"`、三种用法的代码示例（简写名注释、custom `bundle` 对象与 `ssr` 条件、独立包导出 `bundle` 并用包名指定）、"webpack 或 rspack 需显式安装"那句；`### server.builder` 的默认 `"@nuxt/nitro-server"` 与内部用途警告）
- https://nuxt.com/docs/4.x/guide/going-further/builders（builder 的定义句与三个官方构建器、"separates what to build from how to build it"、builder 三项职责、"可部署服务端由 Nitro 经 `@nuxt/nitro-server` 产出、builder 通过类型化契约交付输出"、只有三个官方标识符被按名字识别那句）
