title: Vue3响应式原理
speaker: 信息组-戚付涛
plugins:
    - echarts

<slide class="bg-black-blue aligncenter" image="https://source.unsplash.com/C1HhAQrbykQ/ .dark">

# Vue3响应式原理 {.text-landing.text-shadow}

By 信息组-戚付涛 {.text-intro}

[:fa-github: Github](https://github.com/){.button.ghost}

<slide :class="size-30 aligncenter">

### 深入响应式系统

---

Vue 最标志性的功能就是其低侵入性的响应式系统。组件状态都是由响应式的 JavaScript 对象组成的。当更改它们时，视图会随即自动更新。这让状态管理更加简单直观，但理解它是如何工作的也是很重要的，这可以帮助我们避免一些常见的陷阱

<slide :class="size-50 aligncenter">

## 什么是响应性？

<slide :class="size-50">
## 什么是响应性?

这个术语在今天的各种编程讨论中经常出现，但人们说它的时候究竟是想表达什么意思呢？本质上，响应性是一种可以使我们声明式地处理变化的编程范式。一个经常被拿来当作典型例子的用例即是 Excel 表格：

|  | A | B |
| :----------- | :------------: | ------------: |
| 0   |   1   |     |
| 1    |    2    |       |
| 2   |   3   |     |

这里单元格 A2 中的值是通过公式 = A0 + A1 来定义的 (你可以在 A2 上点击来查看或编辑该公式)，因此最终得到的值为 3，正如所料。但如果你试着更改 A0 或 A1，你会注意到 A2 也随即自动更新了。

<slide :class="size-40 aligncenter">

### 用 JavaScript 写类似的逻辑

---

```javascript {.animated.fadeInRight}
let A0 = 1
let A1 = 2
let A2 = A0 + A1

console.log(A2) // 3

A0 = 2
console.log(A2) // 仍然是 3
```

结论：`当我们更改 A0 后，A2 不会自动更新。`(subscriber)。{.lightSpeedIn.animated.slow}

<slide :class="size-80">

:::column {.vertical-align}
### **如何在JavaScript中实现？**

为了能重新运行计算的代码来更新 A2，我们需要将其包装为一个函数update,然后，我们需要定义几个术语：{.lightSpeedIn.animated.slow}

1. 这个 **update()** 函数会产生一个**副作用**，或者就简称为**作用** (effect)，因为它会更改程序里的状态。{.lightSpeedIn.animated.slow}

2.  **A0** 和 **A1** 被视为这个作用的**依赖** (dependency)，因为它们的值被用来执行这个作用。因此这次作用也可以说是一个它依赖的**订阅者** (subscriber)。{.lightSpeedIn.animated.slow}

----
```javascript {.animated.fadeInRight}
let A2

function update() {
  A2 = A0 + A1
}
```
:::