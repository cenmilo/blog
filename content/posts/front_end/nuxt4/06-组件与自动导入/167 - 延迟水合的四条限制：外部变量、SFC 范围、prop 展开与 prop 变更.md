---
url: /posts/67ce3ae8ab0a51c8cc6445d1fb41c4a4/
title: 167 - 延迟水合的四条限制：外部变量、SFC 范围、prop 展开与 prop 变更
date: 2026-12-06T11:00:00+08:00
updated: 2026-12-06T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：166 给了七种策略，这一篇给那四条会让人白改半天的限制。它们不在同一页，措辞也不同，但都是官方明写的。**限制一：宏的参数不能用外部变量**——"**To ensure that the compiler correctly recognizes this macro, avoid using external variables. The following approach will prevent the macro from being properly recognized:**"，反例是三行：`const strategy = 'visible'` + `const source = () => import('./components/MyComponent.vue')` + `defineLazyHydrationComponent(strategy, source)`。**限制二：只对 SFC 生效**——"**Currently Nuxt's built-in lazy hydration only works in single-file components (SFCs)**"。**限制三：prop 必须写在模板里、不能展开**——同句的后半"**and requires you to define the prop in the template (rather than spreading an object of props via `v-bind`)**"，紧接一句"**It also does not work with direct imports from `#components`.**"**限制四：prop 一变就立刻水合**——"**Any prop change on a lazily hydrated component will trigger hydration immediately. (e.g., changing a prop on a component with `hydrate-never` will cause it to hydrate)**"。还有一条容易忘的配额："Nuxt provides a range of built-in hydration strategies. **Only one strategy can be used per lazy component.**"官方另给了五条使用守则：优先保住首屏内容（"**Avoid delayed hydration for critical, above-the-fold content**"）、与 `v-if` 的取舍（"When using `v-if=\"false\"` on a lazy component, you might not need delayed hydration. You can just use a normal lazy component."）、**共享状态会连带触发**（"Be mindful of shared state (`v-model`) across multiple components. Updating the model in one component can trigger hydration in all components bound to that model."）、按策略本意选择、以及"**Avoid `hydrate-never` on interactive components**: If a component requires user interaction, it should not be set to never hydrate."

categories:
  - 前端开发

tags:
  - nuxt4
  - 延迟水合
  - 限制
  - 性能
  - 编译器宏

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


延迟水合的四种触发条件我记得很熟，翻车却都在别处：把策略抽成常量、用 `v-bind` 把配置展开、给 `hydrate-never` 的组件改了个 prop。这三种都不是"语法错误"，而是**静默失效**。

## 四条限制分别写在哪儿

| # | 限制 | 出处 | 措辞强度 |
| --- | --- | --- | --- |
| 1 | 宏的两个参数不能用外部变量 | `defineLazyHydrationComponent` 页 Parameters 前的 warning | "avoid using external variables" |
| 2 | 只对 SFC 生效 | 组件目录页的 warning | "only works in single-file components (SFCs)" |
| 3 | prop 必须写在模板里，不能 `v-bind` 展开；`#components` 直接导入也不行 | 同一条 warning 的后半 | "requires you to define the prop in the template" |
| 4 | 任何 prop 变化都会**立即触发水合** | 组件目录页的 note | "will trigger hydration immediately" |

外加一条配额："Nuxt provides a range of built-in hydration strategies. **Only one strategy can be used per lazy component.**"

四条的共性是：**前三条都是"编译器要读你的代码文本"**，第四条是运行时行为。这决定了它们的排查方式完全不同。

## 限制一：策略与来源都得写死在调用里

官方原文与反例：

> To ensure that the compiler correctly recognizes this macro, **avoid using external variables**. The following approach will **prevent the macro from being properly recognized**:

```vue
<script setup lang="ts">
const strategy = 'visible'
const source = () => import('./components/MyComponent.vue')
const LazyHydrationMyComponent = defineLazyHydrationComponent(strategy, source)
</script>
```

这个反例特别值得看，因为它**看起来完全正确**：两个变量类型都对、值也对、甚至 `source` 就是官方要求的 `() => Promise<Component>` 形态。问题只在"它们是变量"。

官方给的可用写法就是把这两个参数**字面地**放回调用里：

```vue
<script setup lang="ts">
const LazyHydrationMyComponent = defineLazyHydrationComponent(
  'visible',
  () => import('./components/MyComponent.vue'),
)
</script>
```

推论很实际：**"把七个组件用同一个工厂函数批量生成"这种写法在宏这里不成立**（163 说宏"参数会被编译期读取"就是这个意思）。想复用只能各处写一遍，或者接受"它退化成一个普通异步组件"。

失效症状：宏没被识别，于是那部分代码不会被转换——组件仍然能渲染，但策略不生效（表现是"照样在首屏就水合了"），**不会报错**。

## 限制二与限制三：SFC、模板字面 prop、不能从 `#components` 导

官方一条 warning 把三件事连在一起写：

> Currently Nuxt's built-in lazy hydration **only works in single-file components (SFCs)**, and **requires you to define the prop in the template (rather than spreading an object of props via `v-bind`)**. It **also does not work with direct imports from `#components`**.

拆成三条检查项：

| 检查 | 不成立的写法 |
| --- | --- |
| 是不是 `.vue` 单文件组件 | 在 `.ts` 里用 render 函数建的组件 |
| prop 是不是直接写在模板标签上 | `<component :is="comp" v-bind="hydrationProps" />` 这种展开 |
| 组件从哪来 | `import { MyComponent } from '#components'` 之后再挂 hydrate prop |

第二条最容易无意犯错：把 hydrate 配置塞进一个对象再展开，看起来很"整洁"。官方括号里点名的正是这种写法。

第三条和 161 的那句形成了闭环——`#components` 是"绕过自动导入"的出口（096、161），**代价就是失去这批编译期能力**。同类"显式导入丢魔法"的限制我在 165 也见过一次（`.client` 后缀对按真实路径导入的组件无效）。这不是巧合：**这些特性都由编译器在识别到特定写法时才注入**。

**这一条对 164 的推荐是个补充理由**：`<script setup>` + 自动导入的 SFC，恰好是唯一同时满足三检查项的写法。

## 限制四：prop 一变就水合，`hydrate-never` 也不例外

这条 note 我认为是四个里最反直觉的，官方连例子都给到点上：

> Any prop change on a lazily hydrated component will **trigger hydration immediately**. (e.g., **changing a prop on a component with `hydrate-never` will cause it to hydrate**)

读三遍的必要性在于：

- "**Any** prop change"——不限哪个 prop，父组件任何一次传值变化都算。
- 后果是"立刻水合"，不是"重新评估策略"。
- 括号里那句意味着 **`hydrate-never` 也不是绝对不动**：只要父级给它的 prop 变了，它照样被水合。

官方在守则里把这条延伸到了共享状态：

> **Shared State:** Be mindful of shared state (`v-model`) across multiple components. Updating the model in one component can trigger hydration in all components bound to that model.

**这一句的实际影响**：如果你给一组组件都设了 `hydrate-on-visible` 之类的策略，让它们绑同一个 `v-model`，那么用户在其中一个里输入，其余的会**因为 prop 变化提前水合**——延迟策略名存实亡。所以我给共享状态的组件用延迟水合前，先确认它们是不是靠 props 传递这份状态。

## 只允许一个策略

> Nuxt provides a range of built-in hydration strategies. **Only one strategy can be used per lazy component.**

所以 `hydrate-on-visible` 与 `hydrate-after` 这种组合不是一套叠加方案（官方没写叠加语义）。要"进视口后再等一会"这类复合条件，可行的做法是**换成 `if` 策略**：把条件自己在 setup 里算出来，喂给 `:hydrate-when`（166 那个例子正是这个形状——`myFunction()` 里手动置 `isReady.value = true`）。这是我按定义推的替代路径，官方没写成一条建议。

## 官方五条守则（我认为最该背的是后两条）

> Delayed hydration can offer performance benefits, but it's essential to use it correctly:
>
> 1. **Prioritize In-Viewport Content:** Avoid delayed hydration for critical, above-the-fold content. It's best suited for content that isn't immediately needed.
> 2. **Conditional Rendering:** When using `v-if="false"` on a lazy component, you might not need delayed hydration. You can just use a normal lazy component.
> 3. **Shared State:** Be mindful of shared state (`v-model`) across multiple components. Updating the model in one component can trigger hydration in all components bound to that model.
> 4. **Use Each Strategy's Intended Use Case:** Each strategy is optimized for a specific purpose.
>    - `hydrate-when` is best for components that might not always need to be hydrated.
>    - `hydrate-after` is for components that can wait a specific amount of time.
>    - `hydrate-on-idle` is for components that can be hydrated when the browser is idle.
> 5. **Avoid `hydrate-never` on interactive components:** If a component requires user interaction, it should not be set to never hydrate.

第 2 条容易被忽略，因为它是**反向建议**：已经有 `v-if="false"` 的话，`Lazy` 前缀本身就够（161 那句"still load eagerly unless conditionally rendered"讲的就是这套配合），再加一层延迟水合是多余。

第 5 条与限制四是一对：**别指望 `hydrate-never` 既能保住静态、又能挡住 prop 变化引起的意外水合**——官方直接建议"需要交互的组件不要设 never"。

## 怎么确认策略真的生效

四个限制里三个是静默失效，所以我的验证顺序是固定的：

1. **绑上 `@hydrated`**（166），控制台一句 log。没有它，"延迟生效"与"根本没生效"看起来一模一样。
2. 用 `hydrate-on-visible` 时，滚动到组件前应该没有 log，进入视口才有一次；`rootMargin` 是否被读，可以用一个明显值（如 `'300px'`）对比滚动距离。
3. 三种失效写法各改一次试：把 `v-bind` 展开换成字面 prop；把 `.ts` 组件换成 `.vue`；把 `#components` 导入换成自动导入的标签名。**每改一次看一次 log 时机**，一次只动一处。
4. 最后确认它没被 prop 变化提前触发：把该组件的绑定 prop 固定成常量再测一次。

这套动作能测出本篇四条限制里的全部四条——第 1、2、3 条改变"有没有 log/何时 log"，第 4 条改变"提前 log"。

## 自查

1. 四条限制分别写在哪个页面？哪三条本质上是同一个原因（编译器读文本）？
2. 为什么官方的反例看起来完全正确却失效？它违反了哪句话？
3. "把七个组件用一个工厂函数批量生成"这种写法能不能用在宏上？依据是什么？
4. 三种不成立的写法分别对应哪一项检查？`.client` 后缀那条相似限制在哪一篇？
5. `hydrate-never` 为什么不是"绝对不动"？官方给的例子是怎么说的？
6. 一组共享 `v-model` 的组件为什么可能提前水合？官方那条守则的措辞是什么？
7. 一个组件能用几个策略？想复合条件时可行的替代是什么？
8. 官方第 2 条守则是反向建议，它建议什么情况下不必用延迟水合？
9. 你的验证步骤里，为什么第一步是绑 `@hydrated`？

## 下一篇

168 讲 `preloadComponents` 与 `prefetchComponents` 这一对：官方明确说后者当前的实现"和前者完全一样"，以及那句话里"We are working to improve this behavior"该怎么读。
