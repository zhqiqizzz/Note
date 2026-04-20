**核心本质**：两者都是 ES6 引入的键值对集合，但**WeakMap 是专门为解决内存泄漏问题设计的特殊 Map**，所有差异都源于「键的引用强度不同」。
## 一、核心维度对比表

|维度|Map|WeakMap|
|---|---|---|
|**键的类型**|支持**任意类型**（原始值 + 对象 + Symbol）|**仅支持对象 / 函数作为键**（null 除外）|
|**键的引用强度**|**强引用**（核心区别）|**弱引用**|
|**垃圾回收行为**|键对象即使无其他外部引用，也不会被 GC 回收|键对象无其他强引用时，会被自动 GC 回收，对应键值对同步消失|
|**可迭代性**|完全支持：`for...of`、`keys()`、`values()`、`entries()`|**完全不可迭代**（无任何遍历方法）|
|**API 支持**|完整 API：`get/set/has/delete/clear` + 遍历方法 + `size` 属性|最小 API：仅 `get/set/has/delete`（无 `clear`、无 `size`）|
|**性能特点**|增删查 O (1)，适合通用键值存储|无遍历开销，GC 友好，适合临时对象关联数据|
|**可序列化**|可通过 `JSON.stringify` 间接序列化|完全不可序列化|

## 二、最关键差异：垃圾回收机制详解

这是 WeakMap 存在的唯一理由，也是最容易被误解的点。

### Map 的强引用内存泄漏问题

```javascript
// 演示：Map 导致的内存泄漏
let obj = { name: "大数据对象", data: new Array(1000000).fill(0) };
const map = new Map();

map.set(obj, "关联数据");
obj = null; // 切断外部引用

// ❌ obj 永远不会被垃圾回收！
// 因为 map 还持有对它的强引用
console.log(map.size); // 1
console.log(map.keys().next().value); // { name: "大数据对象", ... }
```

**后果**：如果这个 Map 是全局的，或者生命周期很长，那么这个大对象会一直占用内存，直到页面关闭。

### WeakMap 的弱引用自动回收


```javascript
// 演示：WeakMap 的自动垃圾回收
let obj = { name: "大数据对象", data: new Array(1000000).fill(0) };
const weakmap = new WeakMap();

weakmap.set(obj, "关联数据");
obj = null; // 切断外部引用

// ✅ obj 会被垃圾回收，对应键值对自动消失
// 注意：GC 时机由 JS 引擎决定，不是立即执行
// 此时 weakmap 中已无该键值对
```

**原理**：WeakMap 对键的引用是 "弱" 的，不会被垃圾回收器计入引用计数。当键对象没有其他强引用时，会被正常回收，WeakMap 会自动删除对应的键值对。

## 三、为什么 WeakMap 没有遍历和 size？

这是设计上的必然选择，不是技术限制：

- 垃圾回收是**不确定的**（由 JS 引擎在空闲时执行）
- 如果允许遍历 WeakMap，遍历结果会随时变化，无法预测
- 为了保证程序的确定性，ES 标准直接禁止了 WeakMap 的遍历能力

> [!important]
> 💡 重要误区：WeakMap 的**键是弱引用，但值不是**
> 
> ```javascript
> const weakmap = new WeakMap();
> let key = {};
> let value = { bigData: new Array(1000000).fill(0) };
> 
> weakmap.set(key, value);
> key = null; // key 被回收，value 也会被回收（因为键值对一起消失）
> ```

## 四、各自的典型使用场景

### Map 适用场景（通用键值存储）

1. 需要遍历键值对的场景
2. 键可以是原始值（字符串、数字、布尔值）的场景
3. 需要持久化存储键值对的场景
4. 实现缓存、字典、哈希表等通用数据结构

```javascript
// 通用缓存示例
const cache = new Map();
function getCachedData(id) {
  if (cache.has(id)) return cache.get(id);
  const data = fetchDataFromDB(id);
  cache.set(id, data);
  return data;
}
```

### WeakMap 适用场景（核心：对象关联临时数据）

这是 WeakMap 唯一正确的使用方式，其他场景都应该用 Map。

#### 1. 给 DOM 元素附加私有数据（最常用）

```javascript
// 给按钮元素存储点击次数
const clickCount = new WeakMap();
const btn = document.querySelector("button");

clickCount.set(btn, 0);
btn.addEventListener("click", () => {
  const count = clickCount.get(btn) + 1;
  clickCount.set(btn, count);
  console.log(`点击了 ${count} 次`);
});

// ✅ 当 btn 从 DOM 中移除时，clickCount 中的对应数据会自动被 GC 回收
// 不需要手动删除，完全避免内存泄漏
```

#### 2. 实现类的真正私有属性

```javascript
// 利用 WeakMap 实现私有属性，外部无法访问
const privateData = new WeakMap();

class User {
  constructor(name, age) {
    // 每个实例对应 WeakMap 中的一个键
    privateData.set(this, { name, age });
  }

  getName() {
    return privateData.get(this).name;
  }

  getAge() {
    return privateData.get(this).age;
  }
}

const user = new User("张三", 20);
console.log(user.getName()); // 张三
console.log(user.name); // undefined（无法直接访问私有属性）

// ✅ 当 user 实例被销毁时，privateData 中的对应数据会自动回收
```

#### 3. Vue3 响应式原理的核心实现

Vue3 的 `reactive` API 就是用 WeakMap 存储对象的响应式依赖：

```javascript
// 简化版 Vue3 响应式原理
const targetMap = new WeakMap();

function reactive(target) {
  return new Proxy(target, {
    get(target, key) {
      // 收集依赖
      track(target, key);
      return target[key];
    },
    set(target, key, value) {
      target[key] = value;
      // 触发依赖更新
      trigger(target, key);
      return true;
    }
  });
}

// ✅ 当响应式对象被销毁时，targetMap 中的对应依赖会自动回收
// 这是 Vue3 比 Vue2 内存管理更优秀的重要原因之一
```

#### 4. 缓存与对象生命周期绑定的数据

比如缓存组件实例的计算结果，组件销毁时缓存自动清空，不需要手动清理。

## 五、ES2024 新增 WeakMap 扩展

ES2024 为 WeakMap 新增了两个实用方法，弥补了部分 API 缺失：

- `WeakMap.prototype.addAll(entries)`：批量添加多个键值对
- `WeakMap.prototype.deleteAll(keys)`：批量删除多个键值对

```javascript
const weakmap = new WeakMap();
const obj1 = {}, obj2 = {}, obj3 = {};

// 批量添加
weakmap.addAll([
  [obj1, "value1"],
  [obj2, "value2"],
  [obj3, "value3"]
]);

// 批量删除
weakmap.deleteAll([obj1, obj3]);

console.log(weakmap.has(obj2)); // true
console.log(weakmap.has(obj1)); // false
```

## 六、最佳实践与常见误区

### ✅ 最佳实践

1. **优先使用 Map**：除非明确需要 WeakMap 的内存回收特性，否则都用 Map
2. **WeakMap 只用于对象关联数据**：不要用 WeakMap 做通用键值存储
3. **不要依赖 WeakMap 的 GC 时机**：垃圾回收是不确定的，不要在业务逻辑中依赖它
4. **Vue/React 组件中优先使用 WeakMap**：存储组件实例相关的临时数据，避免内存泄漏

### ❌ 常见误区

1. ❌ **WeakMap 可以用 Map 替代**
    
    错误。在需要自动内存回收的场景，Map 会导致内存泄漏，无法替代 WeakMap。
    
2. ❌ **WeakMap 的值也是弱引用**
    
    错误。只有键是弱引用，值是强引用。如果值引用了键，仍然会导致内存泄漏。
    
3. ❌ **WeakMap 可以存储原始值作为键**
    
    错误。原始值没有引用计数的概念，不能作为 WeakMap 的键。
    
4. ❌ **WeakMap 比 Map 性能更好**
    
    错误。在增删查操作上，两者性能几乎没有差异。WeakMap 的优势只在于内存管理。


## 总结

- **Map**：通用的键值对集合，适合绝大多数场景
- **WeakMap**：特殊的键值对集合，专门用于**给对象附加临时数据**，自动避免内存泄漏
- **核心判断标准**：如果键的生命周期和值的生命周期应该一致，就用 WeakMap；否则用 Map