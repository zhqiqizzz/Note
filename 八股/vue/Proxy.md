## 一、Proxy 是什么？

Proxy 是 ES6（ES2015）新增的**原生内置对象**，你可以把它理解成**目标对象的 “代理层”**—— 外界对目标对象的**所有操作**（读属性、写属性、删属性、调用函数等），都必须先经过这层代理。

### 核心作用

通过 Proxy，你可以**拦截并改写这些操作的默认行为**，比如：

- 读取属性时，偷偷加个默认值；
- 写入属性时，做个数据校验；
- 删除属性时，判断一下有没有权限；
- 甚至可以 “监控” 对象的一举一动，自动触发其他操作（比如更新页面）。

## 二、核心语法

创建 Proxy 的语法非常简单：

```javascript
const proxy = new Proxy(target, handler);
```

### 三个关键概念

1. **target（目标对象）**：被代理的对象，可以是普通对象、数组、函数、Class 实例，甚至另一个 Proxy。
2. **handler（处理器对象）**：用来定义 “拦截规则” 的对象，里面包含各种拦截函数（叫 “陷阱 Trap”）。
3. **proxy（代理对象）**：`new Proxy()` 返回的新对象 ——**只有操作 proxy，才会触发拦截**，直接操作 target 会绕过代理。

### 最简入门示例

先看一个最基础的例子，直观感受拦截效果：

```javascript
// 1. 目标对象
const target = { name: "张三", age: 20 };

// 2. 处理器对象：定义拦截规则
const handler = {
  // 拦截“读取属性”操作
  get(target, prop) {
    console.log(`正在读取属性：${prop}`);
    return target[prop]; // 返回目标对象的属性值
  },

  // 拦截“写入属性”操作
  set(target, prop, value) {
    console.log(`正在修改属性：${prop}，新值：${value}`);
    target[prop] = value; // 修改目标对象的属性
    return true; // 严格模式下必须返回布尔值，表示修改成功
  }
};

// 3. 创建代理对象
const proxy = new Proxy(target, handler);

// 4. 操作代理对象（必须操作 proxy 才会触发拦截）
console.log(proxy.name); 
// 输出：正在读取属性：name → 张三

proxy.age = 25; 
// 输出：正在修改属性：age，新值：25
```

## 三、5 个最常用的拦截器（Trap）

Proxy 总计支持 **13 种拦截器**，覆盖对象几乎所有操作，这里重点讲开发中最常用的 5 个，每个配实用例子。

### 1. `get(target, prop, receiver)`：拦截属性读取

**触发场景**：读取代理对象的属性时（比如 `proxy.name`、`proxy['age']`）。

**核心用途**：设置属性默认值、私有属性保护、属性计算、依赖收集。

#### 例子：属性默认值

```javascript
const obj = { name: "李四" };
const proxy = new Proxy(obj, {
  get(target, prop) {
    // 如果属性存在，返回原值；不存在，返回默认值
    return prop in target ? target[prop] : "该属性不存在";
  }
});

console.log(proxy.name); // 李四
console.log(proxy.age);  // 该属性不存在
```

### 2. `set(target, prop, value, receiver)`：拦截属性赋值

**触发场景**：给代理对象的属性赋值时（比如 `proxy.age = 30`）。

**核心用途**：数据校验、数据变更监听、类型约束。

#### 例子：数据校验

```javascript
const user = { age: 20 };
const userProxy = new Proxy(user, {
  set(target, prop, value) {
    if (prop === "age") {
      // 校验年龄必须是 0-120 的整数
      if (!Number.isInteger(value) || value < 0 || value > 120) {
        throw new Error("年龄必须是 0-120 的整数");
      }
    }
    target[prop] = value;
    return true;
  }
});

userProxy.age = 25; // 正常
userProxy.age = 130; // 抛出错误：年龄必须是 0-120 的整数
```

### 3. `deleteProperty(target, prop)`：拦截属性删除

**触发场景**：用 `delete` 删除代理对象的属性时（比如 `delete proxy.name`）。

**核心用途**：删除权限控制、禁止删除核心属性。

#### 例子：禁止删除核心属性

```javascript
const user = { id: 1, name: "王五" };
const userProxy = new Proxy(user, {
  deleteProperty(target, prop) {
    if (prop === "id") {
      throw new Error("id 是核心属性，禁止删除");
    }
    delete target[prop];
    return true;
  }
});

delete userProxy.name; // 正常删除
delete userProxy.id;   // 抛出错误：id 是核心属性，禁止删除
```

### 4. `has(target, prop)`：拦截 `in` 运算符

**触发场景**：用 `in` 判断属性是否存在时（比如 `'name' in proxy`）。

**核心用途**：隐藏内部私有属性（比如约定下划线开头的属性是私有的）。

#### 例子：隐藏私有属性

```javascript
const user = { name: "赵六", _password: "123456" };
const userProxy = new Proxy(user, {
  has(target, prop) {
    // 如果是下划线开头的私有属性，假装不存在
    if (prop.startsWith("_")) {
      return false;
    }
    return prop in target;
  }
});

console.log("name" in userProxy);      // true
console.log("_password" in userProxy); // false（假装不存在）
```

### 5. `apply(target, thisArg, args)`：拦截函数调用

**触发场景**：调用代理的函数 / 方法时（比如 `proxyFn()`、`proxyFn.call(...)`）。

**核心用途**：参数校验、函数节流防抖、调用日志、异常捕获。

#### 例子：函数调用日志

```javascript
function add(a, b) {
  return a + b;
}

const addProxy = new Proxy(add, {
  apply(target, thisArg, args) {
    console.log(`调用函数 ${target.name}，参数：${args}`);
    const result = target.apply(thisArg, args);
    console.log(`函数执行结果：${result}`);
    return result;
  }
});

addProxy(1, 2);
// 输出：
// 调用函数 add，参数：1,2
// 函数执行结果：3
```

## 四、Proxy 的 4 个经典实战场景
### 1. 响应式系统（Vue 3 核心原理）

#### 副作用函数
##### 1. 大白话解释

“副作用” 就是：**你做了一件事（比如修改数据），除了这件事本身，还对外部产生了其他影响**。

举个生活中的例子：

- 你按了一下空调遥控器（操作数据：把 “开关” 属性改成 “开”）。
- 除了遥控器上的灯亮了（这件事本身），空调开始制冷了（额外影响）—— 这就是副作用。
##### 2. 代码例子理解

先看**没有副作用**的函数（纯函数）：

```javascript
// 纯函数：输入相同，输出一定相同，不影响外部
function add(a, b) {
  return a + b;
}
add(1, 2); // 永远返回 3，外面的世界没变化
```

再看**有副作用**的函数：

```javascript
// 有副作用的函数：修改了外部的变量
let count = 0;
function increaseCount() {
  count++; // 修改了外部的 count
  console.log(`count 变成了：${count}`); // 打印日志，也是对外部的影响
}
increaseCount(); // 执行后，外部的 count 变了
```

在前端开发中，**最常见的副作用就是 “更新页面视图”**—— 比如你修改了数据，页面上的文字跟着变了，这就是副作用。

#### 响应式

用 Proxy 拦截数据的读写，数据变化时自动更新页面（结合 “副作用函数”）：

```html
<!-- 页面上的元素 -->
<div id="app">
  名字：<span id="nameSpan">张三</span>
</div>

<script>
  // 1. 目标对象：存储数据
  const user = {
    name: "张三"
  };

  // 2. 副作用函数：更新页面视图
  function updateView() {
    document.getElementById("nameSpan").textContent = user.name;
    console.log("页面已更新！");
  }

  // 3. 用 Proxy 代理 user
  const userProxy = new Proxy(user, {
    // 拦截“写属性”操作
    set(target, prop, value) {
      target[prop] = value; // 先把数据改了
      updateView(); // 数据改完后，自动执行副作用函数（更新页面）
      return true;
    }
  });

  // 4. 测试：修改代理对象的数据
  setTimeout(() => {
    userProxy.name = "李四"; // 2秒后，页面会自动变成“李四”
  }, 2000);
</script>
```
#### 这里的逻辑

1. 我们定义了一个副作用函数 `updateView()`，它的作用是更新页面。
2. 用 Proxy 代理 `user` 对象，拦截 “写属性” 操作。
3. 一旦有人修改 `userProxy.name`，Proxy 就会先修改数据，然后自动调用 `updateView()` 更新页面。

这就是**响应式的雏形**—— 数据变了，视图自动更新。

### 2. 私有属性保护

结合 `get`、`set`、`deleteProperty`、`has`，完整保护私有属性：

```javascript
const user = { name: "小明", _password: "abc123" };
const userProxy = new Proxy(user, {
  // 禁止读取私有属性
  get(target, prop) {
    if (prop.startsWith("_")) throw new Error("无权访问私有属性");
    return target[prop];
  },
  // 禁止修改私有属性
  set(target, prop, value) {
    if (prop.startsWith("_")) throw new Error("无权修改私有属性");
    target[prop] = value;
    return true;
  },
  // 禁止删除私有属性
  deleteProperty(target, prop) {
    if (prop.startsWith("_")) throw new Error("无权删除私有属性");
    delete target[prop];
    return true;
  },
  // 隐藏私有属性
  has(target, prop) {
    return prop.startsWith("_") ? false : prop in target;
  }
});
```

### 3. 缓存代理

拦截函数调用，对结果做缓存，重复调用相同参数时直接返回缓存：

```javascript
function expensiveCalc(num) {
  console.log("执行复杂计算...");
  return num * 2;
}

const calcProxy = new Proxy(expensiveCalc, {
  cache: new Map(), // 用 Map 存储缓存
  apply(target, thisArg, args) {
    const key = args.join(",");
    if (this.cache.has(key)) {
      console.log("命中缓存！");
      return this.cache.get(key);
    }
    const result = target.apply(thisArg, args);
    this.cache.set(key, result);
    return result;
  }
});

calcProxy(10); // 执行复杂计算... → 20
calcProxy(10); // 命中缓存！ → 20
```

### 4. 数组操作拦截

Proxy 可以原生完美支持数组的所有操作（下标修改、`push`、`pop`、`length` 变更等），这是 `Object.defineProperty` 做不到的：

```javascript
const arr = [1, 2, 3];
const arrProxy = new Proxy(arr, {
  get(target, prop) {
    console.log(`读取数组属性：${prop}`);
    return target[prop];
  },
  set(target, prop, value) {
    console.log(`修改数组属性：${prop}，新值：${value}`);
    target[prop] = value;
    return true;
  }
});

arrProxy[0] = 10;   // 修改数组属性：0，新值：10
arrProxy.push(4);    // 会依次读取 push、length，然后修改 length 和下标 3
console.log(arrProxy.length); // 读取数组属性：length → 4
```

## 五、关键注意事项

### 1. `this` 指向问题

代理对象的方法中，`this` 默认指向 **proxy 代理实例**，而非原始 target 对象。如果 target 内部依赖 `this`，可能出问题，需用 `Reflect` 转发操作保证 `this` 正确：

```javascript
const target = {
  name: "张三",
  sayName() {
    console.log(this.name); // 这里的 this 会指向 proxy
  }
};
const proxy = new Proxy(target, {
  get(target, prop, receiver) {
    // 用 Reflect.get 转发，保证 this 指向正确
    return Reflect.get(target, prop, receiver);
  }
});
proxy.sayName(); // 张三
```

### 2. 可撤销代理

用 `Proxy.revocable()` 可以创建可撤销的代理，调用 `revoke()` 后代理立即失效：

```javascript
const target = { name: "李四" };
const { proxy, revoke } = Proxy.revocable(target, {
  get(target, prop) {
    return target[prop];
  }
});

console.log(proxy.name); // 李四
revoke(); // 撤销代理
console.log(proxy.name); // 抛出 TypeError：代理已失效
```

### 3. 只能代理一层对象

Proxy 只能代理一层对象，无法直接拦截深层嵌套对象的操作。解决方案是在 `get` 中递归创建代理：

```javascript
function deepProxy(target) {
  if (typeof target !== "object" || target === null) return target;
  return new Proxy(target, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      return deepProxy(value); // 递归代理深层对象
    },
    set(target, prop, value, receiver) {
      return Reflect.set(target, prop, value, receiver);
    }
  });
}

const data = deepProxy({ user: { name: "王五" } });
data.user.name = "赵六"; // 可以拦截深层属性修改
```

## 总结

Proxy 是 ES6 提供的强大元编程工具，核心是 **“拦截对象操作，改写默认行为”**。通过常用的 5 个拦截器，你可以实现数据校验、私有属性保护、缓存代理、响应式系统等高级功能。