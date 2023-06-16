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

## Vue 3 中的使用

当我们在学习 Vue 3 的时候，可以通过一个简单示例，看看什么是 Vue 3 中的响应式：

```html
<!-- HTML 内容 -->
<div id="app">
    <div>Price: {{price}}</div>
    <div>Total: {{price * quantity}}</div>
    <div>getTotal: {{getTotal}}</div>
</div>
```

```javascript
const app = Vue.createApp({ // ① 创建 APP 实例
    data() {
        return {
            price: 10,
            quantity: 2
        }
    },
    computed: {
        getTotal() {
            return this.price * this.quantity * 1.1
        }
    }
})
app.mount('#app')  // ② 挂载 APP 实例
```

当我们修改 price 或 quantity 值的时候，页面上引用它们的地方，内容也能正常展示变化后的结果。

<slide :class="size-50 aligncenter">
通过创建 APP 实例和挂载 APP 实例即可，这时可以看到页面中分别显示对应数值：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bccaca7ec584466c90ca06f3e960048a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

<slide :class="size-50 aligncenter">

## 普通js中如何实现？

```javascript
let price = 10, quantity = 2;
const total = price * quantity;
console.log(`total: ${total}`); // total: 20
price = 20;
console.log(`total: ${total}`); // total: 20
```

从这可以看出，在修改 price 变量的值后， total 的值并没有发生改变。

**那么如何修改上面代码，让 total 能够自动更新呢？**

<slide :class="size-100">

:::column {.vertical-align}


- ① 初始化一个 Set 类型的 dep 变量，用来存放需要执行的副作用（ effect 函数），这边是修改 total 值的方法；
- ② 创建 track() 函数，用来将需要执行的副作用保存到 dep 变量中（也称收集副作用）；
- ③ 创建 trigger() 函数，用来执行 dep 变量中的所有副作用；
在每次修改 price 或 quantity 后，调用 trigger() 函数执行所有副作用后， total 值将自动更新为最新值。
----
我们其实可以将修改 total 值的方法保存起来，等到与 total 值相关的变量（如 price 或 quantity 变量的值）发生变化时，触发该方法，更新 total 即可。我们可以这么实现：

```javascript
let price = 10, quantity = 2, total = 0;
const dep = new Set(); // ① 
const effect = () => { total = price * quantity };
const track = () => { dep.add(effect) };  // ②
const trigger = () => { dep.forEach( effect => effect() )};  // ③

track();
console.log(`total: ${total}`); // total: 0
trigger();
console.log(`total: ${total}`); // total: 20
price = 20;
trigger();
console.log(`total: ${total}`); // total: 40
```
:::

<slide :class="size-50 aligncenter">

## 实现单个对象的响应式

通常，我们的对象具有多个属性，并且每个属性都需要自己的 dep。我们如何存储这些？比如：

```javascript
let product = { price: 10, quantity: 2 };
```

从前面介绍我们知道，我们将所有副作用保存在一个 Set 集合中，而该集合不会有重复项，这里我们引入一个 Map 类型集合（即 depsMap ），其 key 为对象的属性（如： price 属性）， value 为前面保存副作用的 Set 集合（如： dep 对象）。

<slide :class="size-100">

:::column {.vertical-align}

- ① 初始化一个 Map 类型的 depsMap 变量，用来保存每个需要响应式变化的对象属性（key 为对象的属性， value 为前面 Set 集合）；
- ② 创建 track() 函数，用来将需要执行的副作用保存到 depsMap 变量中对应的对象属性下（也称收集副作用）；
- ③ 创建 trigger() 函数，用来执行 dep 变量中指定对象属性的所有副作用；
这样就实现监听对象的响应式变化，在 product 对象中的属性值发生变化， total 值也会跟着更新。

----

```javascript
let product = { price: 10, quantity: 2 }, total = 0;
const depsMap = new Map(); // ① 
const effect = () => { total = product.price * product.quantity };
const track = key => {     // ②
  let dep = depsMap.get(key);
  if(!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  dep.add(effect);
}

const trigger = key => {  // ③
  let dep = depsMap.get(key);
  if(dep) {
    dep.forEach( effect => effect() );
  }
};

track('price');
console.log(`total: ${total}`); // total: 0
effect();
console.log(`total: ${total}`); // total: 20
product.price = 20;
trigger('price');
console.log(`total: ${total}`); // total: 40
```

:::


<slide :class="size-50 aligncenter">

##  实现多个对象的响应式

如果我们有多个响应式数据，比如同时需要观察对象 a 和对象 b  的数据，那么又要如何跟踪每个响应变化的对象？

这里我们引入一个 WeakMap 类型的对象，将需要观察的对象作为 key ，值为前面用来保存对象属性的 Map 变量。

<slide :class="size-100">

:::column

- ① 初始化一个 WeakMap 类型的 targetMap 变量，用来要观察每个响应式对象；
- ② 创建 track() 函数，用来将需要执行的副作用保存到指定对象（ target ）的依赖中（也称收集副作用）；
- ③ 创建 trigger() 函数，用来执行指定对象（ target ）中指定属性（ key ）的所有副作用；
这样就实现监听对象的响应式变化，在 product 对象中的属性值发生变化， total 值也会跟着更新。

----

```javascript
let product = { price: 10, quantity: 2 }, total = 0;
const targetMap = new WeakMap();     // ① 初始化 targetMap，保存观察对象
const effect = () => { total = product.price * product.quantity };
const track = (target, key) => {     // ② 收集依赖
  let depsMap = targetMap.get(target);
  if(!depsMap){
    targetMap.set(target, (depsMap = new Map()));
  }
  let dep = depsMap.get(key);
  if(!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  dep.add(effect);
}

const trigger = (target, key) => {  // ③ 执行指定对象的指定属性的所有副作用
  const depsMap = targetMap.get(target);
  if(!depsMap) return;
    let dep = depsMap.get(key);
  if(dep) {
    dep.forEach( effect => effect() );
  }
};

track(product, 'price');
console.log(`total: ${total}`); // total: 0
effect();
console.log(`total: ${total}`); // total: 20
product.price = 20;
trigger(product, 'price');
console.log(`total: ${total}`); // total: 40
```
:::


<slide :class="size-50 aligncenter">

在前面，我们介绍了如何在数据发生变化后，自动更新数据，但存在的问题是，每次需要手动通过触发 track() 函数搜集依赖，通过 trigger() 函数执行所有副作用，达到数据更新目的。

**那么有没有什么办法能够自动实现依赖收集和执行副作用呢？**

<slide :class="size-50 aligncenter">

这里我们引入 JS 对象访问器的概念，解决办法如下：

---

- 在读取（GET 操作）数据时，自动执行 track() 函数自动收集依赖；
- 在修改（SET 操作）数据时，自动执行 trigger() 函数执行所有副作用；

那么如何拦截 GET 和 SET 操作？接下来看看 Vue2 和 Vue3 是如何实现的：

---
- 在 Vue2 中，使用 ES5 的 Object.defineProperty() 函数实现；
- 在 Vue3 中，使用 ES6 的 Proxy 和 Reflect API 实现；

需要注意的是：Vue3 使用的 Proxy 和 Reflect API 并不支持 IE。
Object.defineProperty() 函数这边就不多做介绍，可以阅读文档，下文将主要介绍 Proxy 和 Reflect API。


<slide :class="size-50 aligncenter">

## 如何使用 Reflect?

通常我们有三种方法读取一个对象的属性：

---

- 使用 . 操作符：leo.name ；
- 使用 [] ： leo['name'] ；
- 使用 Reflect API： Reflect.get(leo, 'name') 。

这三种方式输出结果相同。

<slide :class="size-50 aligncenter">

## 如何使用 Proxy?

Proxy 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。语法如下：

```javascript
const p = new Proxy(target, handler)
```

参数如下：

- **target** 要使用 Proxy 包装的目标对象（可以是任何类型的对象，包括原生数组，函数，甚至另一个代理）。
- **handler** 一个通常以函数作为属性的对象，各属性中的函数分别定义了在执行各种操作时代理 p 的行为。

<slide :class="size-50 aligncenter">

使用 Proxy 方法来改造我们的代码：

```javascript
let product = { price: 10, quantity: 2 };
let proxiedProduct = new Proxy(product, {
  get(target, key, receiver){
    console.log('正在读取的数据：',key);
    return Reflect.get(target, key, receiver);
  },
  set(target, key, value, receiver){
    console.log('正在修改的数据：', key, ',值为：', value);
    return Reflect.set(target, key, value, receiver);
  }
})
proxiedProduct.price = 20;
console.log(proxiedProduct.price); 
// 正在修改的数据： price ,值为： 20
// 正在读取的数据： price
// 20
```
这样便完成 get 和 set 函数来拦截对象的读取和修改的操作。

<slide :class="size-50 aligncenter">

为了方便对比 Vue 3 源码，我们将上面代码抽象一层，使它看起来更像 Vue3 源码：

```javascript
function reactive(target){
  const handler = {  // ① 封装统一处理函数对象
    get(target, key, receiver){
      console.log('正在读取的数据：',key);
      return Reflect.get(target, key, receiver);
    },
    set(target, key, value, receiver){
      console.log('正在修改的数据：', key, ',值为：', value);
      return Reflect.set(target, key, value, receiver);
    }
  }
  
  return new Proxy(target, handler); // ② 统一调用 Proxy API
}

let product = reactive({price: 10, quantity: 2}); // ③ 将对象转换为响应式对象
product.price = 20;
console.log(product.price); 
// 正在修改的数据： price ,值为： 20
// 正在读取的数据： price
// 20
```

<slide :class="size-50 aligncenter">

## 修改 track 和 trigger 函数

通过上面代码，我们已经实现一个简单 reactive() 函数，用来将普通对象转换为响应式对象。但是还缺少自动执行 track() 函数和 trigger() 函数，接下来修改上面代码：

```javascript
const targetMap = new WeakMap();
let total = 0;
const effect = () => { total = product.price * product.quantity };
const track = (target, key) => { 
  let depsMap = targetMap.get(target);
  if(!depsMap){
    targetMap.set(target, (depsMap = new Map()));
  }
  let dep = depsMap.get(key);
  if(!dep) {
    depsMap.set(key, (dep = new Set()));
  }
  dep.add(effect);
}

const trigger = (target, key) => {
  const depsMap = targetMap.get(target);
  if(!depsMap) return;
    let dep = depsMap.get(key);
  if(dep) {
    dep.forEach( effect => effect() );
  }
};

const reactive = (target) => {
  const handler = {
    get(target, key, receiver){
      console.log('正在读取的数据：',key);
      const result = Reflect.get(target, key, receiver);
      track(target, key);  // 自动调用 track 方法收集依赖
      return result;
    },
    set(target, key, value, receiver){
      console.log('正在修改的数据：', key, ',值为：', value);
      const oldValue = target[key];
      const result = Reflect.set(target, key, value, receiver);
      if(oldValue != result){
         trigger(target, key);  // 自动调用 trigger 方法执行依赖
      }
      return result;
    }
  }
  
  return new Proxy(target, handler);
}

let product = reactive({price: 10, quantity: 2}); 
effect();
console.log(total); 
product.price = 20;
console.log(total); 
// 正在读取的数据： price
// 正在读取的数据： quantity
// 20
// 正在修改的数据： price ,值为： 20
// 正在读取的数据： price
// 正在读取的数据： quantity
// 40
```

<slide :class="size-50 aligncenter">

## activeEffect 和 ref

在上面的代码中，还存在一个问题： track 函数中的依赖（ effect 函数）是外部定义的，当依赖发生变化， track 函数收集依赖时都要手动修改其依赖的方法名。

比如现在的依赖为 foo 函数，就要修改 track 函数的逻辑，可能是这样

```javascript
const foo = () => { /**/ };
const track = (target, key) => {     // ②
  // ...
  dep.add(foo);
}
```

**那么如何解决这个问题呢？**

<slide :class="size-50 aligncenter">

1. 引入 activeEffect 变量

接下来引入 activeEffect 变量，来保存当前运行的 effect 函数。

```javascript
let activeEffect = null;
const effect = eff => {
  activeEffect = eff; // 1. 将 eff 函数赋值给 activeEffect
  activeEffect();     // 2. 执行 activeEffect
  activeEffect = null;// 3. 重置 activeEffect
}
```

然后在 track 函数中将 activeEffect 变量作为依赖：

```javascript
const track = (target, key) => {
    if (activeEffect) {  // 1. 判断当前是否有 activeEffect
        let depsMap = targetMap.get(target);
        if (!depsMap) {
            targetMap.set(target, (depsMap = new Map()));
        }
        let dep = depsMap.get(key);
        if (!dep) {
            depsMap.set(key, (dep = new Set()));
        }
        dep.add(activeEffect);  // 2. 添加 activeEffect 依赖
    }
}
```

使用方式修改为：

```javascript
effect(() => {
    total = product.price * product.quantity
});
```

这样就可以解决手动修改依赖的问题，这也是 Vue3 解决该问题的方法。

<slide :class="size-50 aligncenter">

```javascript
const targetMap = new WeakMap();
let activeEffect = null; // 引入 activeEffect 变量

const effect = eff => {
  activeEffect = eff; // 1. 将副作用赋值给 activeEffect
  activeEffect();     // 2. 执行 activeEffect
  activeEffect = null;// 3. 重置 activeEffect
}

const track = (target, key) => {
    if (activeEffect) {  // 1. 判断当前是否有 activeEffect
        let depsMap = targetMap.get(target);
        if (!depsMap) {
            targetMap.set(target, (depsMap = new Map()));
        }
        let dep = depsMap.get(key);
        if (!dep) {
            depsMap.set(key, (dep = new Set()));
        }
        dep.add(activeEffect);  // 2. 添加 activeEffect 依赖
    }
}

const trigger = (target, key) => {
    const depsMap = targetMap.get(target);
    if (!depsMap) return;
    let dep = depsMap.get(key);
    if (dep) {
        dep.forEach(effect => effect());
    }
};

const reactive = (target) => {
    const handler = {
        get(target, key, receiver) {
            const result = Reflect.get(target, key, receiver);
            track(target, key);
            return result;
        },
        set(target, key, value, receiver) {
            const oldValue = target[key];
            const result = Reflect.set(target, key, value, receiver);
            if (oldValue != result) {
                trigger(target, key);
            }
            return result;
        }
    }

    return new Proxy(target, handler);
}

let product = reactive({ price: 10, quantity: 2 });
let total = 0, salePrice = 0;
// 修改 effect 使用方式，将副作用作为参数传给 effect 方法
effect(() => {
    total = product.price * product.quantity
});
effect(() => {
    salePrice = product.price * 0.9
});
console.log(total, salePrice);  // 20 9
product.quantity = 5;
console.log(total, salePrice);  // 50 9
product.price = 20;
console.log(total, salePrice);  // 100 18
```

<slide :class="size-50 aligncenter">




## 引入 ref 方法

熟悉 Vue3 Composition API 的朋友可能会想到 Ref，它接收一个值，并返回一个响应式可变的 Ref 对象，其值可以通过 value 属性获取。

> ref：接受一个内部值并返回一个响应式且可变的 ref 对象。ref 对象具有指向内部值的单个 property .value。

```javascript
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

<slide :class="size-50 aligncenter">
我们有 2 种方法实现 ref 函数：

---

- 使用 rective 函数

```javascript
const ref = intialValue => reactive({value: intialValue});
```

这样是可以的，虽然 Vue3 不是这么实现。

- 使用对象的属性访问器（计算属性）

属性方式去包括：getter 和 setter。
```javascript
const ref = raw => {
  const r = {
    get value(){
      track(r, 'value');
      return raw;
    },
    
    set value(newVal){
    	raw = newVal;
      trigger(r, 'value');
    }
  }
  return r;
}
```

<slide :class="size-50 aligncenter">

## 总结

本次分享带大家从头开始学习如何实现简单版 Vue 3 响应式，实现了 Vue3 Reactivity 中的核心方法（ effect / track / trigger / reactive 等方法），帮助大家了解其核心，提高项目开发效率和代码调试能力。

<slide class="bg-black-blue aligncenter" image="https://cn.bing.com/az/hprichbg/rb/PragueChristmas_EN-AU8649790921_1920x1080.jpg .dark">

## 谢谢您的聆听！ {.animated.tada}
