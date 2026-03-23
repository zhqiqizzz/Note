## 一、先搞懂基础 `effect`：响应式系统的 “心脏”

`effect` 是 Vue 3 响应式系统的**最底层核心 API**（你可以理解为 “副作用函数的注册器”），所有的响应式逻辑（包括 `renderEffect`、`watch`、`computed`）都是基于它实现的。

### 1. `effect` 是什么？

简单来说，`effect` 就是**用来注册 “响应式副作用函数” 的工具**—— 它会把你传入的函数包装成一个 “响应式副作用”，让这个函数：

- 执行时会自动**收集依赖**（记录自己依赖了哪些响应式数据）；
- 当依赖的数据变化时，会自动**重新执行**。

### 2. `effect` 的核心逻辑（伪代码）

我们用极简伪代码还原 `effect` 的核心结构，帮你理解它和 `track`/`trigger` 的关系：

```javascript
// 全局变量：临时保存当前正在执行的副作用函数
let activeEffect = null;

// effect 函数：注册副作用
function effect(fn, options = {}) {
  // 1. 把原始函数包装成响应式副作用函数
  const effectFn = () => {
    // 2. 执行前，把当前副作用赋值给全局变量 activeEffect
    activeEffect = effectFn;
    // 3. 执行原始函数（比如 render 函数、watch 回调）
    // 执行过程中会读取响应式数据，触发 track 收集依赖
    const res = fn();
    // 4. 执行完，清空 activeEffect
    activeEffect = null;
    return res;
  };

  // 5. 把 options 挂载到 effectFn 上（比如 scheduler）
  effectFn.options = options;
  // 6. 把依赖集合挂载到 effectFn 上（用于清理依赖）
  effectFn.deps = [];

  // 7. 如果不是惰性的（默认不是），立即执行一次
  if (!options.lazy) {
    effectFn();
  }

  // 8. 返回包装后的副作用函数
  return effectFn;
}
```

### 3. 基础 `effect` 的简单例子

我们用一个手动创建的 `effect` 看效果：

```javascript
import { reactive, effect } from 'vue';

const state = reactive({ count: 0 });

// 注册副作用函数
effect(() => {
  console.log('count 当前值：', state.count);
});

// 修改数据，副作用自动重新执行
state.count++; // 输出：count 当前值：1
```

## 二、`effect` 的核心参数：`scheduler`（调度器）

这是 `renderEffect` 最关键的参数 ——**没有 `scheduler`，就没有 Vue 3 的异步更新队列，页面会频繁卡顿**。

### 1. 为什么需要 `scheduler`？

如果没有 `scheduler`，每次数据变化都会立即执行 `effectFn`，比如：

```javascript
// 没有 scheduler 的情况：每次修改都立即执行
state.count++;
state.count++;
state.count++;
// 会输出 3 次，执行 3 次 render，更新 3 次 DOM——太浪费了！
```

我们希望：**多次数据变化合并成一次更新**，等所有数据修改完，再统一执行一次 `effectFn`，这就需要 `scheduler`。

### 2. `scheduler` 的作用

`scheduler` 是一个函数，当你给 `effect` 传了 `scheduler` 后：

- 数据变化时，**不会立即执行 `effectFn`**，而是会执行 `scheduler` 函数；
- 你可以在 `scheduler` 里控制 `effectFn` 什么时候执行（比如放到异步队列里，等同步代码执行完再统一执行）。

### 3. `scheduler` 的简单例子

```javascript
import { reactive, effect } from 'vue';

const state = reactive({ count: 0 });

// 注册副作用，传 scheduler
effect(
  () => {
    console.log('count 当前值：', state.count);
  },
  {
    // scheduler：把 effectFn 放到 setTimeout 里异步执行
    scheduler: (effectFn) => {
      setTimeout(effectFn, 0);
    }
  }
);

// 连续修改 3 次
state.count++;
state.count++;
state.count++;
console.log('同步代码执行完了');

// 输出顺序：
// 同步代码执行完了
// count 当前值：3（只执行了 1 次！）
```

## 三、`renderEffect` 是怎么基于 `effect` 实现的？

对于一个 Vue 组件来说，**负责 “模板渲染→虚拟 DOM→真实 DOM” 这个核心流程的副作用函数，就是 Vue 内部为该组件创建的专属 `renderEffect`，且每个组件只有一个这样的核心渲染副作用**。

它不是 “全局固定” 的（每个组件有自己的 `renderEffect`），但对单个组件而言，它是 “固定且唯一” 的 —— 从组件初始化到销毁，始终是这个副作用函数负责核心渲染逻辑。

### 1. `renderEffect` 的 “固定性” 体现在哪里？

- **创建时机固定**：组件初始化阶段（`mount` 阶段），Vue 会为组件创建这个 `renderEffect` 并立即执行一次，完成首次渲染；
- **逻辑固定**：这个副作用函数的核心逻辑永远是「执行 `render` 函数生成新 VNode → 调用 `patch` 做 Diff → 更新真实 DOM」，不会变；
- **销毁时机固定**：组件卸载时，这个 `renderEffect` 会被停止（`effect.stop()`），不再响应数据变化。

### 2. 伪代码还原 `renderEffect` 的创建与执行

```javascript
// Vue 内部组件挂载逻辑（简化版）
function mountComponent(vnode, container) {
  const component = {
    vnode, // 保存当前虚拟 DOM
    render: compiledRenderFunction, // 模板编译后的 render 函数
    renderEffect: null // 核心渲染副作用函数
  };

  // 1. 创建专属的 renderEffect（固定用于组件渲染）
  component.renderEffect = effect(
    () => {
      // 副作用函数的核心逻辑（固定不变）
      const newVNode = component.render(); // 执行 render 生成新 VNode
      if (component.vnode) {
        // 数据更新时：对比新旧 VNode，Diff 后更新真实 DOM
        patch(component.vnode, newVNode, container);
      } else {
        // 首次渲染：直接把 VNode 挂载成真实 DOM
        patch(null, newVNode, container);
      }
      component.vnode = newVNode; // 保存新 VNode 供下次对比
    },
    {
      // 调度器：让更新异步执行（Vue 3 的异步更新队列）
      scheduler: queueJob
    }
  );
}
```

### 3. 为什么 `renderEffect` 能 “响应数据变化”？

- 首次执行 `renderEffect` 时，会调用 `component.render()`，而 `render` 函数里会读取响应式数据（比如 `count.value`）；
- 读取数据时触发 `track`，把 `renderEffect` 收集为该数据的依赖；
- 数据变化时触发 `trigger`，只会执行这个组件对应的 `renderEffect`，而非全局所有副作用 —— 这也是 Vue 3 性能优化的关键（精准更新）。

### 4. `renderEffect` 的两个核心特点

#### （1）核心逻辑固定：执行 `render` + `patch`

`renderEffect` 的副作用逻辑永远是：

- 执行组件的 `render` 函数 → 生成新虚拟 DOM；
- 调用 `patch` 函数（内部是 Diff 算法）→ 对比新旧虚拟 DOM，更新真实 DOM。

#### （2）特殊的 `scheduler`：`queueJob`（异步更新队列）

Vue 没有用 `setTimeout`，而是自己实现了一个更高效的**异步更新队列**，`queueJob` 就是把 `renderEffect` 放到这个队列里的函数：

- 多次数据变化时，同一个组件的 `renderEffect` 只会被放到队列里一次；
- 等当前同步代码执行完（微任务阶段），再统一执行队列里的所有 `renderEffect`；
- 这样就实现了**多次数据变化合并成一次渲染更新**，大幅提升性能。

## 四、其他 “非固定” 的副作用函数

除了核心的 `renderEffect`，页面渲染 / 运行过程中还会出现其他副作用函数，它们不负责核心渲染，但也是响应式系统的一部分，且是 “按需创建、不固定” 的：

### 1. `watch` 对应的副作用函数

你写的 `watch` 会创建专属的副作用函数，只负责监听数据变化执行回调，不参与渲染：

```javascript
import { ref, watch } from 'vue';
const count = ref(0);

// 这个 watch 会创建一个独立的副作用函数（非 renderEffect）
watch(count, (newVal) => {
  console.log('count 变了', newVal);
});
```

- 这个副作用函数只在 `count` 变化时执行回调，不会触发 `render` 或 DOM 更新；
- 每个 `watch` 对应一个独立的副作用函数，是 “按需创建” 的，不是固定的。

### 2. `computed` 对应的副作用函数

`computed` 内部也会创建惰性的副作用函数，只在依赖变化且被访问时执行：

```javascript
import { ref, computed } from 'vue';
const count = ref(0);
const double = computed(() => count.value * 2);

// computed 内部的副作用函数：
// 1. 依赖 count，但默认不执行（惰性）；
// 2. 只有访问 double.value 时才执行；
// 3. count 变化时，只会标记 double 为“脏值”，下次访问才重新计算。
```

- 这个副作用函数只负责计算值，不参与渲染，且是 “惰性执行” 的，和 `renderEffect` 完全独立。

### 3. 手动创建的 `effect` 函数

如果你手动调用 `effect` 创建副作用，也会生成独立的函数：

```javascript
import { ref, effect } from 'vue';
const count = ref(0);

// 手动创建的副作用函数（非 renderEffect）
const customEffect = effect(() => {
  console.log('count 当前值：', count.value);
});
```

- 这个函数会响应 `count` 变化，但不会触发组件渲染，是完全自定义的。

## 五、`renderEffect` 的完整生命周期（配合响应式）

我们把 `renderEffect` 从创建到销毁的完整流程，和响应式系统串起来讲，帮你建立全局认知：

### 1. 阶段一：创建 `renderEffect`（组件挂载 `mount`）

- Vue 初始化组件，拿到模板编译后的 `render` 函数；
- 调用 `effect(renderLogic, { scheduler: queueJob })`，创建 `renderEffect`；
- 因为 `options.lazy` 是 `false`（默认），所以会**立即执行一次 `renderEffect`**。

### 2. 阶段二：首次执行 `renderEffect`（首次渲染 + 依赖收集）

- 执行 `renderEffect` 里的 `render` 函数；
- `render` 函数里会读取响应式数据（比如 `count.value`、`state.name`）；
- 读取数据时，触发 `Proxy` 的 `get` 拦截，调用 `track`；
- `track` 把当前的 `renderEffect` 收集为这些数据的依赖（存入 `targetMap`）；
- `render` 执行完，生成初始虚拟 DOM；
- 调用 `patch(null, newVNode, container)`，把虚拟 DOM 挂载成真实 DOM；
- 页面首次渲染完成！

### 3. 阶段三：数据变化 → 触发更新（`trigger` → `scheduler` → 异步队列）

- 用户修改响应式数据（比如 `count.value++`）；
- 触发 `Proxy` 的 `set` 拦截，调用 `trigger`；
- `trigger` 从 `targetMap` 里找到依赖该数据的 `renderEffect`；
- 因为 `renderEffect` 有 `scheduler`，所以**不会立即执行 `renderEffect`**，而是调用 `scheduler`（`queueJob`）；
- `queueJob` 把 `renderEffect` 放到 Vue 的异步更新队列里（如果已经在队列里了，就不重复放）；
- 等当前同步代码执行完，进入微任务阶段，Vue 统一执行队列里的所有 `renderEffect`。

### 4. 阶段四：重新执行 `renderEffect`（重新渲染 + Diff 更新）

- 异步队列执行 `renderEffect`；
- 再次执行 `render` 函数，生成**新的虚拟 DOM**（因为数据变了）；
- 调用 `patch(oldVNode, newVNode, container)`；
- `patch` 内部用 Diff 算法对比新旧虚拟 DOM，找出最小变化；
- 只更新变化的真实 DOM 节点；
- 页面更新完成！

### 5. 阶段五：销毁 `renderEffect`（组件卸载 `unmount`）

- 组件卸载时，Vue 会调用 `renderEffect.stop()`；
- `stop` 会清理 `renderEffect` 的所有依赖（从 `targetMap` 里移除）；
- 之后数据再变化，也不会触发 `renderEffect` 了；
- `renderEffect` 生命周期结束。