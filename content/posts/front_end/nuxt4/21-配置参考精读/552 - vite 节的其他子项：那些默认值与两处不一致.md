---
url: /posts/2cb886f06113bd1b67f4021a848c4c12/
title: 552 - vite 节的其他子项：那些默认值与两处不一致
date: 2027-04-14T09:00:00+08:00
updated: 2027-04-14T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`vite` 节除 `$client`/`$server` 之外还列了这些子项，官方给了默认值的有：`build.assetsDir` 默认 **`"_nuxt/"`**、`build.emptyOutDir` 默认 `false`、`cacheDir` 默认 `"/<rootDir>/node_modules/.cache/vite"`、`clearScreen` 默认 `true`、`define` 默认五个常量（`__VUE_PROD_HYDRATION_MISMATCH_DETAILS__: false`、`process.dev`/`import.meta.dev`/`process.test`/`import.meta.test` 全是 `false`）、`esbuild` 默认四项（`target: "esnext"`、`jsxFactory: "h"`、`jsxFragment: "Fragment"`、`tsconfigRaw: {}`）、`mode` 默认 **`"production"`**、`optimizeDeps.esbuildOptions` 与 `esbuild` 同形、`optimizeDeps.exclude` 默认 `["vue-demi"]`、`resolve.extensions` 默认七项（含 `.json`）、`root` 默认 `"/<rootDir>"`、`server.fs.allow` 默认四行、`vue.features.propsDestructure` 与 `vue.isProduction` 默认 `true`、`vue.template.transformAssetUrls` 默认五种标签的属性表。两处细节要对号：**`vite.build.assetsDir` 是 `"_nuxt/"`（不带前导斜杠），而 `app.buildAssetsDir` 是 `"/_nuxt/"`（带）**；`vueJsx` 的默认值里出现的是一个 JSON-schema 形状而不是函数，`hoistStatic` 那一行连类型都没有。

categories:
  - 前端开发

tags:
  - nuxt4
  - vite
  - 构建
  - 默认值
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`vite` 这一节很长。想改一个值之前，先看它现在的默认是什么——这一节里已经写了默认值的项有十几条，而其中两条的形状与另一节对不上。

## 有默认值的项，逐条

| 子项 | 类型 | 官方默认 |
| --- | --- | --- |
| `build.assetsDir` | `string` | `"_nuxt/"` |
| `build.emptyOutDir` | `boolean` | `false` |
| `cacheDir` | `string` | `"/<rootDir>/node_modules/.cache/vite"` |
| `clearScreen` | `boolean` | `true` |
| `define` | `object` | 五个常量，全是 `false`（见下） |
| `esbuild` | `object` | `target: "esnext"`、`jsxFactory: "h"`、`jsxFragment: "Fragment"`、`tsconfigRaw: {}` |
| `mode` | `string` | `"production"` |
| `optimizeDeps.esbuildOptions` | `object` | 与 `esbuild` 同一份四项 |
| `optimizeDeps.exclude` | `array` | `["vue-demi"]` |
| `resolve.extensions` | `array` | `[".mjs", ".js", ".ts", ".jsx", ".tsx", ".json", ".vue"]` |
| `root` | `string` | `"/<rootDir>"` |
| `server.fs.allow` | `array` | `["/<rootDir>/.nuxt", "/<rootDir>/app", "/<rootDir>", "/<workspaceDir>"]` |
| `vue.features.propsDestructure` | `boolean` | `true` |
| `vue.isProduction` | `boolean` | `true` |
| `vue.template.compilerOptions` | `object` | 未列 |
| `vue.template.transformAssetUrls` | `object` | 五种标签的属性表（见下） |
| `vueJsx` | `object` | 一个 schema 形状（见下） |

四条读起来最有用的：

**`mode` 默认 `"production"`。** 这一条常被忽略——Vite 的 `mode` 在 Nuxt 的默认值就写着生产模式。要按 mode 分环境加载东西，得先确认这个默认值是否符合你的预期。

**`server.fs.allow` 默认四行。** 允许被 dev server 读的范围就是 `.nuxt`、`app`、项目根、workspace 根这四条。workspace 之外（比如 monorepo 里另一个包的路径）默认**不在这四条内**。

**`cacheDir` 落在 `node_modules/.cache/vite`。** `nuxt cleanup` 的清除清单里恰好有一项 `node_modules/.vite`——这个目录就是 Vite 缓存的家，改 `cacheDir` 会让清理命令那一项对不上（清理清单是官方写死的那几处）。

**`optimizeDeps.exclude` 默认排除 `vue-demi`。** 这是唯一被官方点名预排除的依赖；要再加排除项，就往这个数组里写。

## define 的五个默认常量

官方给的 JSON 是五行：

```json
{
  "__VUE_PROD_HYDRATION_MISMATCH_DETAILS__": false,
  "process.dev": false,
  "import.meta.dev": false,
  "process.test": false,
  "import.meta.test": false
}
```

四行是 `process.*`／`import.meta.*` 形态的**同名双写**（同一个标志在两种写法下都给值），第一行是 Vue 的编译标志。

`__VUE_PROD_HYDRATION_MISMATCH_DETAILS__` 这个名字直译是"生产环境水合不匹配的细节"。**Nuxt 文档在这里只给了默认 `false`，没有解释打开后会有什么输出**——它是 Vue 侧的标志，效果说明不在这一节。要用水合不匹配的详细信息，就得知道这个默认值是关着的。

## transformAssetUrls 那张表

`vue.template.transformAssetUrls` 的默认值是一份"标签 → 会被当资源处理的属性"表：

```json
{
  "video": ["src", "poster"],
  "source": ["src"],
  "img": ["src"],
  "image": ["xlink:href", "href"],
  "use": ["xlink:href", "href"]
}
```

**这五行解释了模板里 `<img src="...">` 的字面量路径会被重写、而运行时拼出来的值不会**——机制是"只有被静态解析到的属性才进这张表的处理"。同一节在 `#### image`/`#### img`/`#### source`/`#### use`/`#### video` 五个子项里各给了默认数组，形状与这张表逐一对应。

## 两处对不上号的地方

**第一处：`"_nuxt/"` 还是 `"/_nuxt/"`。** `vite.build.assetsDir` 的默认是**不带前导斜杠**的 `"_nuxt/"`，而 `app.buildAssetsDir` 的默认是**带**前导斜杠的 `"/_nuxt/"`。两者官方说明都指向"构建产物里放资源的目录名"，但**这两个键的关系与谁生效，文档没有写**。要改资源目录名，本系列建议改 `app.buildAssetsDir`（它的说明里明确写了"相对 `baseURL`，设了 `cdnURL` 就相对它"），改完务必检查产物里引用是否统一。

**第二处：`vueJsx` 与 `hoistStatic`。** `vueJsx` 的默认 JSON 里出现的是一个带 `$schema`／`title`／`description`／`tags` 的形状（`isCustomElement` 那一项被渲染成了 schema 描述），而不是一眼能读的选项表；`### vue` 组里的 `#### script` → `##### hoistStatic` 那一行**只有标题、没有类型也没有默认值**。

这两处的可读性问题不在你的理解力上。**要设 `vueJsx` 的选项，按 Vite/`@vitejs/plugin-vue-jsx` 那边的选项形状写**（配置页给的 Vue 配置一节也说了 `vite.vue` 与 `vite.vueJsx` 对应这两个插件）；`hoistStatic` 这类没写清的项，**只能自己验证生效与否**。

## 自查

- 有默认值的十余条里最该记的四条：`mode` 默认 `"production"`、`server.fs.allow` 只允许四行、`cacheDir` 在 `node_modules/.cache/vite`、`optimizeDeps.exclude` 默认排除 `vue-demi`。
- `define` 默认五个常量全为 `false`，其中 `__VUE_PROD_HYDRATION_MISMATCH_DETAILS__` 是 Vue 侧标志，Nuxt 未解释其效果。
- `vue.template.transformAssetUrls` 的五行表就是"模板里哪些属性算资源"的答案，也解释了静态字面量会被重写而运行时值不会。
- **`vite.build.assetsDir`（`"_nuxt/"`）与 `app.buildAssetsDir`（`"/_nuxt/"`）前导斜杠不一致，两者关系文档未写**；改资源目录名建议动 `app` 那一个。
- `vueJsx` 的默认值形状不可读、`vue.script.hoistStatic` 没有类型与默认，这两类都要按 Vite 插件侧文档写并自行验证。

## 下一篇

下一篇 553 进 `webpack` 节：这一节比 `vite` 还长，读法要按分类来——输出文件名一组、加载器一组、转换与插件一组，而官方对整节的定性同样只有一句"直接传给 webpack"。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## vite` 节其余子项：`### build` 下 `assetsDir`/`emptyOutDir`、`cacheDir`、`clearScreen`、`define` 五行默认、`esbuild` 四项默认、`mode`、`optimizeDeps.esbuildOptions` 与 `exclude`、`publicDir`、`resolve.extensions` 七项、`root`、`server.fs.allow` 四行、`vue.features.propsDestructure`、`vue.isProduction`、`vue.script.hoistStatic`（无类型无默认）、`vue.template.compilerOptions`、`vue.template.transformAssetUrls` 五项表与五个 `####` 子项、`vueJsx` 的默认形状）
- https://nuxt.com/docs/4.x/getting-started/configuration（Vue Configuration 一节：`vite.vue` 对应 `@vitejs/plugin-vue`、`vite.vueJsx` 对应 `@vitejs/plugin-vue-jsx` 与 `webpack.loaders.vue` 的对应说法）
- https://nuxt.com/docs/4.x/api/nuxt-config 与 https://nuxt.com/docs/4.x/api/commands/cleanup（`app.buildAssetsDir` 默认 `"/_nuxt/"` 及其相对 `baseURL`/`cdnURL` 的说明；清除清单里的 `node_modules/.vite`）
