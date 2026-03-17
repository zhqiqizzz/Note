## 一、先搞懂：为什么需要虚拟 DOM？

在讲虚拟 DOM 之前，我们先想一个问题：**为什么不直接操作真实 DOM，而是要绕一圈用虚拟 DOM？**

### 1. 真实 DOM 操作很慢

真实 DOM 是浏览器提供的复杂对象，属性非常多（一个简单的 `div` 就有几百个属性），操作真实 DOM 的成本很高：

- 每次修改真实 DOM，浏览器都要重新计算布局（重排）、重新绘制（重绘）；
- 频繁操作真实 DOM，页面会卡顿。

### 2. 虚拟 DOM 是什么？

虚拟 DOM（Virtual DOM，简称 VNode）就是**用普通的 JS 对象来描述真实 DOM 的结构**。因为 JS 对象的操作非常快，我们可以先在 JS 层面对比新旧虚拟 DOM，找出最小的变化，再只更新这部分真实 DOM，从而减少重排重绘。

### 3. 虚拟 DOM 的简单例子

比如真实 DOM 是这样的：

```html
<div id="app" class="container">
  <p>Hello Vue</p>
</div>
```

对应的虚拟 DOM（简化版）就是这样的 JS 对象：

```javascript
const vnode = {
  type: 'div', // 标签名
  props: { id: 'app', class: 'container' }, // 属性
  children: [ // 子节点
    {
      type: 'p',
      children: 'Hello Vue'
    }
  ]
};
```

---

## 二、再讲 render 函数：生成虚拟 DOM 的 “工厂”

Vue 的模板（`<template>`）最终会被编译成 **render 函数**，执行 render 函数，就会返回一棵虚拟 DOM 树。
### 1. 模板 → render 函数

Vue 3 内部会把你写的模板，编译成 render 函数。比如：

```html
<!-- 你写的模板 -->
<template>
  <div id="app">
    <p>{{ message }}</p>
    <button @click="count++">点击: {{ count }}</button>
  </div>
</template>
```

编译后生成的 render 函数（简化版）大概长这样：

```javascript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createBlock as _createBlock } from "vue";

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", { id: "app" }, [
    _createVNode("p", null, _toDisplayString(_ctx.message)),
    _createVNode("button", {
      onClick: $event => (_ctx.count++)
    }, "点击: " + _toDisplayString(_ctx.count))
  ]));
}
```

### 2. `h` 函数：手动创建虚拟节点

Vue 3 提供了 `h` 函数（全称 `createVNode`），用来手动创建虚拟节点。上面 render 函数里的 `_createVNode` 就是 `h` 函数。

我们可以用 `h` 函数手动写一个 render 函数：

```javascript
import { h, ref } from 'vue';

export default {
  setup() {
    const count = ref(0);
    return { count };
  },
  render() {
    // 用 h 函数创建虚拟 DOM
    return h('div', { id: 'app' }, [
      h('p', `点击次数: ${this.count}`),
      h('button', {
        onClick: () => this.count++
      }, '点击+1')
    ]);
  }
};
```

### 3. 执行 render 函数 → 虚拟 DOM

当组件渲染时，Vue 会执行 render 函数，返回一棵虚拟 DOM 树，这棵树描述了页面当前应该长什么样。

---

## 三、核心：Diff 算法 —— 对比新旧虚拟 DOM 的 “放大镜”

当响应式数据变化时，会触发副作用函数，**重新执行 render 函数，生成一棵新的虚拟 DOM 树**。

这时就需要 **Diff 算法**了：它的作用是**对比新旧两棵虚拟 DOM 树，找出最小的变化**，然后只把这部分变化更新到真实 DOM 上，而不是全量替换整个页面。

### 1. Diff 算法的核心前提（为了快）

Diff 算法的复杂度如果是 O (n³)（对比所有节点的所有组合），那页面会卡死。所以 Vue 3 的 Diff 算法做了三个核心假设，把复杂度降到了 O (n)：

1. **只同层对比**：不跨层级对比节点，只对比同一父节点下的子节点；
2. **标签不同直接替换**：如果新旧节点的标签名（`type`）不同，直接销毁旧节点，创建新节点，不继续对比子节点；
3. **标签相同对比属性和子节点**：如果标签名相同，就对比属性和子节点，尽量复用旧节点。

### 2. Vue 3 的核心优化：静态标记（PatchFlags）

Vue 3 在编译阶段就给虚拟节点打上了**静态标记（PatchFlags）**，告诉 Diff 算法：“这个节点只有哪些部分是动态的，其他的不用对比”。

比如之前的模板，编译后的虚拟节点会带标记：

```javascript
// 简化版带标记的虚拟 DOM
const oldVNode = {
  type: 'div',
  props: { id: 'app' }, // 静态属性，不用对比
  children: [
    {
      type: 'p',
      children: '点击次数: 0',
      patchFlag: 1 // PatchFlags.TEXT：只有文本是动态的
    },
    {
      type: 'button',
      props: { onClick: fn },
      children: '点击+1',
      patchFlag: 1 | 16 // TEXT + EVENT：文本和事件是动态的
    }
  ]
};
```

有了静态标记，Diff 算法就不用全量对比了，只需要对比标记的部分，这就是**靶向更新**，比 Vue 2 的全量 Diff 快很多！

### 3. Diff 算法的核心逻辑（简单版）

我们用一个简单的例子，看 Diff 算法是怎么对比的：
#### 例子：新旧虚拟 DOM 对比

```javascript
// 旧虚拟 DOM
const oldVNode = {
  type: 'div',
  children: [
    { type: 'p', key: 'a', children: 'A' },
    { type: 'p', key: 'b', children: 'B' },
    { type: 'p', key: 'c', children: 'C' }
  ]
};

// 新虚拟 DOM（只是把顺序换了，加了一个 D）
const newVNode = {
  type: 'div',
  children: [
    { type: 'p', key: 'a', children: 'A' },
    { type: 'p', key: 'd', children: 'D' },
    { type: 'p', key: 'b', children: 'B' },
    { type: 'p', key: 'c', children: 'C' }
  ]
};
```

#### Diff 对比步骤（核心是「尽量复用节点」）

1. **同层对比根节点**：新旧根节点都是 `div`，标签相同，继续对比子节点。
2. **对比子节点（用 key 快速复用）**：
    
    - Vue 3 会用 `key` 来快速匹配新旧子节点（这就是为什么 `v-for` 必须加 `key`，而且不能用 index）；
    - 旧子节点的 key 是 `a`、`b`、`c`，新子节点的 key 是 `a`、`d`、`b`、`c`；
    - `a`、`b`、`c` 的 key 都存在，直接复用这些节点，只需要调整位置；
    - `d` 的 key 不存在，创建新节点。
    
3. **更新属性和文本**：对于复用的节点，对比静态标记的部分（比如文本），如果有变化就更新。
4. **更新真实 DOM**：只做最小的操作 —— 创建 `d` 节点，调整 `b`、`c` 的位置，其他节点不动。

---

## 四、完整流程串讲：响应式 → render → 虚拟 DOM → Diff → 真实 DOM

现在我们把之前讲的「响应式系统」、「render 函数」、「虚拟 DOM」、「Diff 算法」全部串起来，看 Vue 3 完整的更新流程：

### 1. 初始化阶段

1. 用 `reactive`/`ref` 创建响应式数据；
2. Vue 把模板编译成 render 函数；
3. 执行 render 函数，生成**旧虚拟 DOM 树**；
4. 根据旧虚拟 DOM 树，创建真实 DOM，挂载到页面上；
5. 执行 render 函数时，读取响应式数据，触发「依赖收集」（记录 “渲染函数依赖了这些数据”）。

### 2. 数据变化阶段

1. 用户修改响应式数据（比如 `count++`）；
2. 触发 Proxy 的 `set` 拦截，调用 `trigger`；
3. 找到依赖该数据的副作用函数（也就是「重新执行 render 函数」的函数）；
4. 重新执行 render 函数，生成**新虚拟 DOM 树**。

### 3. Diff 对比阶段

1. 用 Diff 算法对比新旧两棵虚拟 DOM 树；
2. 利用静态标记，只对比动态部分；
3. 利用 `key` 快速复用节点；
4. 找出最小的变化，生成「补丁」（比如 “创建节点”、“修改文本”、“调整位置”）。

### 4. 更新真实 DOM 阶段

1. 根据「补丁」，只更新需要变化的真实 DOM；
2. 浏览器完成重排重绘，页面更新。

---

## 五、总结：为什么这套流程能提升性能？

1. **虚拟 DOM**：用 JS 对象操作替代频繁的真实 DOM 操作，JS 操作很快；
2. **Diff 算法**：对比新旧虚拟 DOM，找出最小变化，只更新这部分真实 DOM，减少重排重绘；
3. **Vue 3 优化**：静态标记（靶向更新）、`key` 复用节点，让 Diff 算法更快。