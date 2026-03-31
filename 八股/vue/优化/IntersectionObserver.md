**IntersectionObserver** 是现代浏览器提供的一个高性能 API，用于**异步监听目标元素与其祖先元素（或视口）的交叉状态**。

简单来说，它能告诉你：“某个元素什么时候进入了屏幕可视区域”或“什么时候离开了”。

它是解决**滚动监听性能问题**的终极方案，彻底替代了传统的 `window.addEventListener('scroll', ...)` + `getBoundingClientRect()` 的计算方式。

### 1. 为什么需要它？（痛点 vs 解决方案）

#### 传统做法

要判断一个元素是否可见，传统做法是监听 `scroll` 事件，然后在回调里疯狂计算：

```javascript
// 性能糟糕的做法
window.addEventListener('scroll', () => {
  const rect = element.getBoundingClientRect();
  const isVisible = (rect.top >= 0 && rect.bottom <= window.innerHeight);
  // scroll 事件每秒可能触发几十到上百次，每次都要计算布局，极易导致掉帧
});
```

**缺点**：

- **主线程阻塞**：`getBoundingClientRect()` 会强制浏览器同步重排（Reflow），消耗巨大。
- **高频触发**：滚动时事件触发过于频繁，即使加了节流（throttle），依然不够流畅。

#### IntersectionObserver 做法

- **异步通知**：浏览器底层（C++层）自动检测交叉状态，只有状态变化时才触发回调。
- **零布局抖动**：不需要手动计算坐标，不触发重排。
- **配置灵活**：可以精确控制“进入多少比例”才算可见。

### 2. 核心用法详解

#### 基本语法

```javascript
const observer = new IntersectionObserver((entries, observer) => {
  entries.forEach(entry => {
    // entry 是每个被观察元素的状态对象
    if (entry.isIntersecting) {
      console.log('元素进入视口啦！');
      // 执行逻辑：加载图片、播放动画、停止视频等
      
      // 常见操作：一旦触发，停止观察（只执行一次）
      observer.unobserve(entry.target); 
    } else {
      console.log('元素离开视口了');
    }
  });
}, options);

// 开始观察某个 DOM 元素
observer.observe(targetElement);

// 停止观察
observer.unobserve(targetElement);

// 完全断开连接（销毁）
observer.disconnect();
```

#### 关键配置项 (Options)

第二个参数 `options` 决定了观察的行为：

```javascript
const options = {
  root: null,           // 默认为视口 (viewport)。如果设为某个父元素，则相对于该父元素检测。
  rootMargin: '0px',    // 根元素的外边距。例如 '100px' 表示元素进入视口 100px 前就触发（预加载）。
  threshold: [0, 0.5, 1] // 阈值数组。
  // 0: 元素刚露出一丁点就触发
  // 0.5: 元素显示 50% 时触发
  // 1: 元素完全显示时触发
};
```
### 3. Vue 3 中的最佳实践

在 Vue 3 中，我们通常将其封装为 **Composable (Hooks)** 或 **自定义指令**，以便复用。

#### 场景 A：封装为 Composable (`useIntersectionObserver`)

适用于需要在组件逻辑中精细控制观察行为的场景（如：无限滚动、数据预加载）。

```javascript
// composables/useIntersectionObserver.js
import { ref, onMounted, onUnmounted } from 'vue';

export function useIntersectionObserver(targetRef, callback, options = {}) {
  let observer = null;

  onMounted(() => {
    const target = targetRef.value;
    if (!target) return;

    observer = new IntersectionObserver((entries) => {
      entries.forEach((entry) => {
        // 将 entry 信息传给回调，方便组件内部处理
        callback(entry, observer);
      });
    }, {
      root: options.root || null,
      rootMargin: options.rootMargin || '0px',
      threshold: options.threshold || 0
    });

    observer.observe(target);
  });

  onUnmounted(() => {
    if (observer) {
      observer.disconnect(); // 组件销毁时务必清理，防止内存泄漏
    }
  });

  return { observer };
}
```

**使用示例：图片懒加载**

```vue
<script setup>
import { ref } from 'vue';
import { useIntersectionObserver } from '@/composables/useIntersectionObserver';

const imgRef = ref(null);
const src = ref(''); // 初始为空，不加载
const realSrc = 'https://example.com/heavy-image.jpg';

const handleIntersect = (entry, observer) => {
  if (entry.isIntersecting) {
    src.value = realSrc; // 进入视口，赋值真实链接开始加载
    observer.unobserve(entry.target); // 加载后停止观察
  }
};

useIntersectionObserver(imgRef, handleIntersect, { threshold: 0.1 });
</script>

<template>
  <div style="height: 1000px;">占位内容，滚动下来</div>
  
  <!-- 初始只显示低清图或 loading -->
  <img ref="imgRef" :src="src || 'loading.gif'" alt="Lazy Image" />
  
  <div style="height: 1000px;">更多内容</div>
</template>
```

#### 场景 B：封装为自定义指令 (`v-observe`)

适用于模板驱动的场景，代码更简洁（如：滚动到某处触发动画类名）。

```javascript
// directives/observe.js
export const vObserve = {
  mounted(el, binding) {
    const { value, modifiers, arg } = binding;
    
    // 支持传入配置对象，或使用默认值
    const options = {
      root: null,
      rootMargin: '0px',
      threshold: modifiers.once ? 0 : 0.5 // 如果有 .once 修饰符，阈值设为 0
    };

    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          // 执行传入的函数
          if (typeof value === 'function') {
            value(entry, observer);
          }
          
          // 如果有 .once 修饰符，触发一次后取消观察
          if (modifiers.once) {
            observer.unobserve(el);
          }
        }
      });
    }, options);

    observer.observe(el);
    
    // 将 observer 挂载到元素上，方便卸载时访问
    el._observer = observer;
  },
  unmounted(el) {
    if (el._observer) {
      el._observer.disconnect();
    }
  }
};
```

**使用示例：**

```vue
<template>
  <!-- 当元素进入视口，执行 fadeIn 函数 -->
  <div v-observe.once="fadeIn">
    我会淡入显示
  </div>
</template>

<script setup>
const fadeIn = (entry, observer) => {
  entry.target.classList.add('animate-fade-in');
};
</script>
```
### 4. 经典应用场景

|场景|传统做法痛点|IntersectionObserver 优势|
|---|---|---|
|**图片/视频懒加载**|需计算所有图片位置，浪费流量和内存|仅加载可见区域，节省带宽，提升首屏速度|
|**无限滚动 (Infinite Scroll)**|需监听 scroll 计算距离底部多远|在列表底部放一个“哨兵元素”，观察到它即加载下一页|
|**滚动动画 (Scroll Reveal)**|频繁修改 class 导致重排|元素进入视口才添加 `animate` 类，性能极高|
|**广告曝光统计**|难以精确判断广告是否真的被用户看到|可设置 `threshold: 0.5`，只有展示超过 50% 才算曝光|
|**吸顶导航 (Sticky Header)**|需高频计算 offsetTop|观察一个“哨兵元素”，当哨兵不可见时，导航变 fixed|
### 5. 注意事项与兼容性

1. **IE 不支持**：
    
    - IntersectionObserver 在 IE 全系列中**不可用**。
    - **解决方案**：如果必须支持 IE，需要引入 polyfill（如 `intersection-observer` npm 包），或者降级为传统的 scroll 监听方案。
    - _注：2026 年的今天，绝大多数现代项目已不再考虑 IE 支持。_
2. **回调是异步的**：
    
    - 它的回调不会在元素插入 DOM 的瞬间立即执行，而是由浏览器在下一帧空闲时触发。如果需要同步知道状态，需自行处理。
3. **内存泄漏风险**：
    
    - **务必**在组件卸载 (`onUnmounted` 或 `unmounted` 钩子) 时调用 `observer.disconnect()` 或 `observer.unobserve()`。否则，单页应用（SPA）路由切换多次后，后台会残留大量观察者，导致内存暴涨。
4. **`rootMargin` 的妙用**：
    
    - 做“预加载”时，设置 `rootMargin: '200px'`。这意味着元素**距离视口还有 200px** 时就会触发回调，让用户感觉不到加载过程，体验更丝滑。