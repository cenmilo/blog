---
url: /posts/748bfcda775009e579f70f4a736829b7/
title: 预处理器：Sass、Less 与 Stylus 的配置点
date: 2027-02-28T10:00:00+08:00
updated: 2027-02-28T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：预处理器在 Nuxt 里没有专门的配置节——官方的做法是"装包 + 走 Vite 的 css 选项"。三步：① 装对应包（`npm install -D sass` / `less` / `stylus` 三条命令分别对应 Sass&SCSS、Less、Stylus）；② 写在 `app/assets` 目录，然后在 `app.vue` 或布局里用该预处理器自己的语法引（`<style lang="scss">` 里 `@use "~/assets/scss/main.scss";`），或者直接放进 `css` 数组；③ 要注入公共代码（比如带颜色变量的 Sass partial），配 `vite.css.preprocessorOptions.scss.additionalData` 或 `sass` 那一份——**两种语法的值不一样**：SCSS 结尾用 `;`，SASS 用 `\n`。Nuxt 默认用 Vite，改用 webpack 就得去看各 preprocessor loader 文档；另外 Vite 有个实验性的 `preprocessorMaxWorkers: true`（CPU 数减一）可以加速预处理器。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预处理器
  - sass
  - preprocessorOptions
  - additionalData
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

在 `nuxt.config` 里搜"scss"是搜不到配置项的——Nuxt 不给预处理器单独开一节，全部走 Vite 的 css 选项。这一篇把这条路径走通，包括那个 SCSS 与 SASS 写法不通用的细节。

## 第一步：装包

> To use a preprocessor like SCSS, Sass, Less or Stylus, install it first.

官方按三种情形分了三个命令（用哪个包管理器按你项目来）：

```bash [Sass & SCSS]
npm install -D sass
```

```bash [Less]
npm install -D less
```

```bash [Stylus]
npm install -D stylus
```

**没有装包之前，配置怎么改都不会生效。**416 说过 `css` 数组里写 `.scss` 时 Nuxt 会按扩展名选预处理器、但 loader 要自备——缺依赖的报错通常长得像"预处理器不可用"，不是"配置写错"。

## 第二步：写在哪儿、怎么引

官方给的位置仍是 `app/assets` 目录："The natural place to write your stylesheets is the `app/assets` directory."，然后两种引法。

**① 在组件里用预处理器自己的语法引**：

```vue [app/pages/app.vue]
<style lang="scss">
@use "~/assets/scss/main.scss";
</style>
```

**② 放进 `css` 数组**（全局、每页都带）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  css: ['~/assets/scss/main.scss'],
})
```

两种情况下官方都保证同一件事：

> In both cases, the compiled stylesheets will be inlined in the HTML rendered by Nuxt.

编译产物内联进渲出的 HTML。415 讲过这两种引法的作用域差别（组件内引只在引处生效、`css` 数组全局）。

SFC 里直接写预处理器代码也是同一套：`<style lang="scss">` / `lang="sass"` / `lang="less"` / `lang="stylus"`——Vite 对 `.scss`、`.sass`、`.less`、`.styl`、`.stylus` **自带支持、不需要配置**，只要包装了就能用（417）。

## 第三步：要注入公共代码就配 preprocessorOptions

典型需求是每个文件都能拿到一份颜色变量 partial，不用每处手写 `@use`。官方给的是 Vite 的选项：

> If you need to inject code in pre-processed files, like a Sass partial with color variables, you can do so with the Vite preprocessors options.

先在 `app/assets` 里建 partial：

```scss [assets/_colors.scss]
$primary: #49240F;
$secondary: #E4A79D;
```

```sass [assets/_colors.sass]
$primary: #49240F
$secondary: #E4A79D
```

注意这两份**结尾风格不同**（scss 用分号、sass 不用），下面的配置也一样要分开写。

```ts twoslash [SCSS]
export default defineNuxtConfig({
  vite: {
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: '@use "~/assets/_colors.scss" as *;',
        },
      },
    },
  },
})
```

```ts twoslash [SASS]
export default defineNuxtConfig({
  vite: {
    css: {
      preprocessorOptions: {
        sass: {
          additionalData: '@use "~/assets/_colors.sass" as *\n',
        },
      },
    },
  },
})
```

**两处差别照抄官方**：`scss` 那份的 `additionalData` 以 `;` 结尾，`sass` 那份以 `\n` 结尾。这就是"我给 scss 配好了，sass 项目却报错"的成因——两个键各自独立、注入内容也要匹配对应语法，不能共用一份字符串。

## 换构建器就没有这段配置

官方在这里划了一条明确的岔路：

> Nuxt uses Vite by default. If you wish to use webpack instead, refer to each preprocessor loader documentation.

也就是说上面 `vite.css.preprocessorOptions` 这一套**只在默认 Vite 构建器下成立**。改用 webpack（`builder` 选项，088/330 那条线）就要去看各 preprocessor loader 的文档（官方给的是 sass-loader 那页），不是把同样的键搬到 `webpack` 节里。

## 一个实验性的加速项

Vite 提供了一个实验选项加速预处理器编译，官方原样转述：

```ts
export default defineNuxtConfig({
  vite: {
    css: {
      preprocessorMaxWorkers: true, // number of CPUs minus 1
    },
  },
})
```

`true` 的含义按注释是 **CPU 数减一**。官方用 `::note` 提醒这是实验选项，要参考 Vite 文档并提供反馈——所以别把它当稳定 API 规划。

## 自查

- 顺序永远是：**装包（sass/less/stylus）→ 写进 `app/assets` → 用 `lang` 属性引或放进 `css` 数组**；没装包，配置改了也没用。
- 两种引法产物都内联进渲出的 HTML。
- 注入公共 partial 走 `vite.css.preprocessorOptions`，且 `scss` 的 `additionalData` 以 `;` 结尾、`sass` 的以 `\n` 结尾，两份不能共用。
- 这套配置只在默认 Vite 下有效；换 webpack 要看各 loader 文档。
- `preprocessorMaxWorkers: true`（CPU 数减一）是实验选项，参考 Vite 文档。

## 下一篇

下一篇 419 讲 PostCSS：Nuxt 自带、在 `postcss.plugins` 里加插件，官方列出四个预配置插件（postcss-import、postcss-url、autoprefixer、cssnano）与配置节里只有 `order` 和 `plugins` 两个键的实情。

## 本文依据

- "To use a preprocessor like SCSS, Sass, Less or Stylus, install it first."与三条安装命令、`app/assets` 位置、`<style lang="scss">` 里 `@use` 与 `css` 数组两种引法、"the compiled stylesheets will be inlined in the HTML rendered by Nuxt"、Sass partial 注入（`_colors.scss`/`_colors.sass` 两份示例与 `vite.css.preprocessorOptions.scss|sass.additionalData` 两份配置）、"Nuxt uses Vite by default. If you wish to use webpack instead, refer to each preprocessor loader documentation"、`preprocessorMaxWorkers: true`（number of CPUs minus 1）与其实验性 `::note`（`getting-started/styling`）。
