---
url: /posts/68217de4dd8d40ed6249caf900211733/
title: 438 - build.transpile 的四种登记写法
date: 2027-03-07T09:00:00+08:00
updated: 2027-03-07T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：`build.transpile` 是 Nuxt 里少数要手动登记依赖的配置，官方给的写法有四种：包名、函数、字符串、匹配依赖文件名的正则对象。需要登记它的场景文档写在别处——ESM 排错那一节让你把导入就报语法错的库加进来，错误码 E1006 让你把调用 `onPrehydrate()` 的库加进来，模块侧的 `addComponent` 有个 `transpile: 'auto'` 会自动判断路径里有没有 `node_modules/`。函数参数这一项文档只留了一个空括号，全文出现过的字段只有示例里的 `isLegacy`。

categories:
  - 前端开发

tags:
  - nuxt4
  - transpile
  - ESM
  - 构建
  - 报错
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

装了一个新库，`nuxt dev` 起不来，终端里是这么一坨：

```bash [Terminal]
export default {}
^^^^^^

SyntaxError: Unexpected token 'export'
```

这类错的根因在 015 那篇讲过（库给了按 bundler 惯例命名的 ESM 产物，Node 的 ESM 上下文不认）。当时给的解法就落在这里：**把这个库登记进 `build.transpile`**。

这一篇讲这一项怎么写，以及文档里还有哪三处会用到它。

## 官方定义：一句话与四种写法

`build` 节下的 `transpile` 全文是这样：

> If you want to transpile specific dependencies with Babel, you can add them here. Each item in transpile can be a package name, a function, a string or regex object matching the dependency's file name.

类型是 `array`。把那句话拆开，一项里有四种合法元素：

| 写法 | 官方原话 | 什么时候用得上 |
| --- | --- | --- |
| 包名 | a package name | 整个库都要过转译，最直接 |
| 字符串 | a string | 与包名同列，文档未说明两者区别 |
| 正则对象 | regex object matching the dependency's file name | 按**文件名**匹配，不是按包名 |
| 函数 | a function | 条件转译，见下一节 |

注意第三项的限定语是"匹配依赖的文件名"——正则匹配的对象是文件名。想按路径匹配，文档没写它支不支持。

## 函数写法与那个空括号

> You can also use a function to conditionally transpile. The function will receive an object ().

官方这句原文里，括号是空的。**参数对象里有什么字段，这一页没写**。文档里出现过的字段只有一个，在同一项的示例里：

```ts
export default defineNuxtConfig({
  build: {
    transpile: [({ isLegacy }) => isLegacy && 'ky'],
  },
})
```

这行的写法值得慢看一遍：函数返回 `isLegacy && 'ky'`。`isLegacy` 为假时，`&&` 短路返回 `false`，也就是这一项**不产出要转译的东西**；为真时返回字符串 `'ky'`，等价于登记了 `ky` 这个包。条件转译就是靠返回值实现的。

至于 `isLegacy` 是什么条件、还有没有别的字段，**文档未说明**。要在自己项目里确认，最省事的办法是在函数里打印一次收到的对象：

```ts
transpile: [(ctx) => {
  console.log(Object.keys(ctx))
  return false
}]
```

这是可以立刻验证的一件事，不用等文档补。

## 另外三处也会让你写它

**第一处：ESM 排错。** 这一节给的例子是最常见的用法：

```ts
export default defineNuxtConfig({
  build: {
    transpile: ['sample-library'],
  },
})
```

紧跟着一句容易漏的提醒：

> You may find that you *also* need to add other packages that are being imported by these libraries.

也就是说，登记了出问题的那个库还不够，**它内部导入的包可能也要一起登记**。第一次加了一个包仍然报错，先想到这里。

**第二处：错误码 E1006。** 这个错误的描述是 `onPrehydrate()` 没有被 Nuxt 构建流水线转换过——它需要编译期处理、且只在服务端生效，所以当它是从一个 Nuxt 不转译的依赖里调用出来的时候就会报。官方的解法原文一句：

> Add the offending library to `build.transpile` in `nuxt.config.ts` so the build pipeline can process its `onPrehydrate()` call.

**第三处：写模块的时候。** Kit 的上下文示例里，模块作者在钩子里直接改这个数组，并且是**带条件地**改：

```ts
if (nuxt.options.builder === '@nuxt/webpack-builder') {
  nuxt.options.build.transpile ||= []
  nuxt.options.build.transpile.push('xstate')
}
```

这个例子先判构建器再 push，说明"要不要登记"在文档的示例逻辑里跟构建器有关；但**换成别的构建器时这一项的行为差异，文档没写**。同一件事还有第二个入口：`addComponent` 有个 `transpile` 选项，类型 `'auto' | boolean`、默认 `false`，说明是"Transpile specified path using build.transpile. If set to `'auto'`, it will set `transpile: true` if `node_modules/` is in path."——注册组件时让 Nuxt 按路径里有没有 `node_modules/` 自动决定要不要走这一项。

顺带一条历史线索：从 2.x 兼容层（bridge）那份配置示例里能看到 `// Do not transpile modules` 和注释掉的 `transpile: false`，那是旧配置里的开关，不属于 4.x 这一节讲的 `build.transpile`。

## 自查

- 四项合法元素：包名、函数、字符串、匹配文件名的正则；类型是数组。
- 函数用于条件转译，返回 `false` 就不登记，返回字符串就登记；参数对象的字段文档没给全，示例里只出现过 `isLegacy`。
- 被那个库导入的包，可能也要一起登记。
- 三个具体触发点：导入报 `SyntaxError: Unexpected token 'export'` 之类的 ESM 错、E1006 的 `onPrehydrate()` 未被转换、模块里按构建器条件 push 或 `addComponent` 的 `transpile: 'auto'`。

## 下一篇

下一篇 439 讲代码分割：包被切成哪几块、谁决定切。路由级是怎么来的、`Lazy` 前缀和全局组件各自出 chunk 的规则不一样，以及官方那句"关掉代码分割通常只会让首屏下载更多"是针对什么写法说的。
