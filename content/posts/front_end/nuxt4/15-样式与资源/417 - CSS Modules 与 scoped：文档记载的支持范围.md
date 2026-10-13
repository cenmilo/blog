---
url: /posts/63ce81f6aa1baa1107f40751a72186e3/
title: 417 - CSS Modules 与 scoped：文档记载的支持范围
date: 2027-02-28T09:00:00+08:00
updated: 2027-02-28T09:00:00+08:00
author: cmdragon

summary:
  摘要：组件级样式隔离有两个正写在不同文档层面的手段（Vue SFC 提供、Nuxt styling 页原文复述），加上 class/style 绑定与 `v-bind` 共四条。① `scoped` 属性："The scoped attribute allows you to style components in isolation. The styles declared with this attribute will only apply to this component."；② CSS Modules 用 `module` 属性，通过注入的 `$style` 变量访问（`<p :class="$style.red">` 配 `<style module>.red{color:red}</style>`）；③ class 与 style 绑定直接吃 Vue 的四种形态（ref/reactive、computed、数组、对象）；④ 样式块里用 `v-bind(color)` 引用 JS 变量，官方强调这个绑定是**动态**的——变量值变了样式会更新。这一节还有一句定位：SFC 让你在组件文件里直接写 CSS 或预处理器代码，因此**不需要为了好用而采用 CSS-in-JS**；真要用的话社区有 pinceau 这类库。

categories:
  - 前端开发

tags:
  - nuxt4
  - scoped
  - CSS Modules
  - v-bind
  - SFC样式
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

"Nuxt 支持 CSS Modules 吗""scoped 能不能用"——答案都在 styling 那页的 SFC 一节里，而且官方把这两件事当作**Vue SFC 自带的特性**来写，不需要额外配置。这一篇把四种手段与它们的边界列出来。

## 先说定位：SFC 的样式块就是主路

官方对 SFC 样式的态度：

> One of the best things about Vue and SFC is how great it is at naturally dealing with styling. You can directly write CSS or preprocessor code in the style block of your components file, therefore you will have a fantastic developer experience without having to use something like CSS-in-JS.

也就是：**在组件的 `<style>` 块里直接写 CSS 或预处理器代码**就够用，不必为了开发体验去上 CSS-in-JS。真要用的话官方也给了出口——社区有第三方库与模块，例如 pinceau。详细的类型化组件 props 与 emits 边界见 405。

## 手段一：scoped

官方定义：

> The scoped attribute allows you to style components in isolation. The styles declared with this attribute will only apply to this component.

```vue
<template>
  <div class="example">
    hi
  </div>
</template>

<style scoped>
.example {
  color: red;
}
</style>
```

两句对应两件事：**隔离**，以及作用范围只到**这个组件本身**。这是"同名 class 互相污染"的正解。

## 手段二：CSS Modules

官方定义与示例：

> You can use CSS Modules with the module attribute. Access it with the injected `$style` variable.

```vue
<template>
  <p :class="$style.red">
    This should be red
  </p>
</template>

<style module>
.red {
  color: red;
}
</style>
```

要点在访问方式：类名不是字符串，而是**注入的 `$style` 对象上的属性**（`:class="$style.red"`）。和 scoped 的差别是取用方式——scoped 里 class 名照原样写、由编译器保证作用域；module 是把类名换成对象键，天然唯一。

## 手段三：class 与 style 绑定

官方把这块整个交给 Vue SFC 能力："You can leverage Vue SFC features to style your components with class and style attributes."，并给全四种形态：

```vue
<!-- Ref and Reactive -->
<div class="static" :class="{ 'active': isActive, 'text-danger': hasError }" />
<div :class="classObject" />

<!-- Computed -->
<div :class="classObject" />

<!-- Array -->
<div :class="[{ active: isActive }, errorClass]" />

<!-- Style -->
<div :style="{ color: activeColor, fontSize: fontSize + 'px' }" />
<div :style="[baseStyles, overridingStyles]" />
<div :style="styleObject" />
```

四种分别是对象条件、computed 派生对象、数组混合、`style` 内联（可对象可数组）。这一节官方只给形态不排优劣，细节指向 Vue 的 class-and-style 文档。

## 手段四：样式块里用 `v-bind`

> You can reference JavaScript variables and expressions within your style blocks with the v-bind function. The binding will be dynamic, meaning that if the variable value changes, the style will be updated.

```vue
<script setup lang="ts">
const color = ref('red')
</script>

<template>
  <div class="text">
    hello
  </div>
</template>

<style>
.text {
  color: v-bind(color);
}
</style>
```

关键是"dynamic"这句：`v-bind` 在样式块里不是拼一次常量，变量变了样式跟着变。这让"主题色由响应式状态驱动"这类需求不必写行内 style。

## 预处理器在样式块里同样可用

官方明确 Vite 自带支持，无需配置：

> SFC style blocks support preprocessor syntax. Vite comes with built-in support for .scss, .sass, .less, .styl and .stylus files without configuration. You just need to install them first, and they will be available directly in SFC with the lang attribute.

即 `<style lang="scss">` / `lang="sass"` / `lang="less"` / `lang="stylus"` 四种写法；**依赖要先装**，装完就能在 SFC 里用。（`postcss` 也有对应的 `lang="postcss"`，419 讲。）

## 边界：这几条都是 Vue/Vite 的能力

值得说明的是：scoped、CSS Modules、class/style 绑定、`v-bind` 在样式块中——Nuxt 文档都是**复述 Vue SFC 规范**，而不是另立一套；预处理器支持是 Vite 自带的。所以排查这几类问题时，"Nuxt 做了什么"的答案通常是"没额外做什么"，行为跟纯 Vue + Vite 一致。webpack 用户则要看 vue-loader 的文档（官方指了这条路）。

## 自查

- `scoped`＝组件内隔离，样式只作用本组件；`module`＝CSS Modules，通过注入的 `$style` 变量取类名。
- class/style 绑定官方给四种形态（对象条件、computed、数组、style 对象/数组），细节归 Vue 文档。
- 样式块里 `v-bind(color)` 是**动态**绑定，变量变样式变。
- SFC 样式块支持 scss/sass/less/styl/stylus，Vite 自带、**依赖要自装**，用 `lang` 属性启用；postcss 用 `lang="postcss"`。
- 官方定位：直接在 style 块写就够了，不必为体验上 CSS-in-JS；要用有 pinceau 等第三方。
- 这几条都是 Vue/Vite 层能力，Nuxt 没有另立规则。

## 下一篇

下一篇 418 讲预处理器怎么装、装完在哪配：`vite.css.preprocessorOptions` 注入 Sass partial（SCSS 与 SASS 的 `additionalData` 写法不同），以及实验性的 `preprocessorMaxWorkers`。

## 本文依据

- SFC 样式定位句（"without having to use something like CSS-in-JS"、pinceau）、Class And Style Bindings 四种形态与"leverage Vue SFC features"、Dynamic Styles With `v-bind` 及其 dynamic 说明、Scoped Styles 定义与示例、CSS Modules 定义与 `$style` 示例、Preprocessors Support（Vite 内建支持 .scss/.sass/.less/.styl/.stylus、要先装、`lang` 属性）、webpack 用户看 vue-loader（`getting-started/styling`）。
