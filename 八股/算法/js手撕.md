## 1. 深拷贝 (Deep Clone)

**核心逻辑：** 解决引用类型在赋值时“牵一发而动全身”的问题。需要递归处理对象和数组，并注意处理循环引用。

```JavaScript
function deepClone(obj, hash = new WeakMap()) {
  if (obj === null) return null;
  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj);
  // 如果不是复杂数据类型，直接返回
  if (typeof obj !== "object") return obj;

  // 处理循环引用：如果已经拷贝过，直接返回
  if (hash.has(obj)) return hash.get(obj);

  let cloneObj = new obj.constructor();
  hash.set(obj, cloneObj);

  for (let key in obj) {
    if (obj.hasOwnProperty(key)) {
      // 递归拷贝每一层
      cloneObj[key] = deepClone(obj[key], hash);
    }
  }
  return cloneObj;
}
```

- **讲解：** 使用 `WeakMap` 是为了防止对象中存在“自己引用自己”导致的死循环。`obj.constructor()` 可以自动兼容数组 `[]` 或对象 `{}` 的创建。

## 2. 防抖与节流 (Debounce & Throttle)

这是性能优化的必考题。

### 防抖 (Debounce)

**场景：** 搜索框输入、窗口大小调整。只在最后一次触发后的 $N$ 毫秒执行。

```JavaScript
function debounce(fn, delay) {
  let timer = null;
  return function(...args) {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}
```

### 节流 (Throttle)

**场景：** 滚动加载、抢购点击。在规定时间内只执行一次。

```JavaScript
function throttle(fn, delay) {
  let lastTime = 0;
  return function(...args) {
    let now = Date.now();
    if (now - lastTime > delay) {
      fn.apply(this, args);
      lastTime = now;
    }
  };
}
```

## 3. 手写 Promise

### 1. 手写promise

```js
class MyPromise {
  constructor(executor) {
    this.status = 'pending'; // 状态：pending | fulfilled | rejected
    this.value = undefined;  // 成功的值
    this.reason = undefined; // 失败的原因
    this.onResolvedCallbacks = []; // 成功回调的数组（发布订阅）
    this.onRejectedCallbacks = []; // 失败回调的数组

    const resolve = (value) => {
      if (this.status === 'pending') {
        this.status = 'fulfilled';
        this.value = value;
        this.onResolvedCallbacks.forEach(fn => fn()); // 执行所有成功回调
      }
    };

    const reject = (reason) => {
      if (this.status === 'pending') {
        this.status = 'rejected';
        this.reason = reason;
        this.onRejectedCallbacks.forEach(fn => fn()); // 执行所有失败回调
      }
    };

    try {
      executor(resolve, reject); // 立即执行
    } catch (err) {
      reject(err);
    }
  }

  then(onFulfilled, onRejected) {
    // 1. 处理值穿透（如果不传回调，直接把值往后传递）
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : val => val;
    onRejected = typeof onRejected === 'function' ? onRejected : err => { throw err };

    // 2. 返回一个新的 Promise 实现链式调用
    return new MyPromise((resolve, reject) => {
      
      // 封装一个处理函数，用来处理 then 回调的返回值 x
      const handle = (callback, val) => {
        // setTimeout 模拟异步微任务
        setTimeout(() => {
          try {
            const x = callback(val);
            // 极简版处理：如果 x 是 MyPromise 实例，直接调用它的 then；否则直接 resolve
            if (x instanceof MyPromise) {
              x.then(resolve, reject);
            } else {
              resolve(x);
            }
          } catch (err) {
            reject(err);
          }
        });
      };

      // 3. 根据当前状态，决定是立即执行还是存入数组
      if (this.status === 'fulfilled') {
        handle(onFulfilled, this.value);
      } 
      else if (this.status === 'rejected') {
        handle(onRejected, this.reason);
      } 
      else if (this.status === 'pending') {
        // 还没出结果，先把函数存起来（注意这里也要包一层 handle）
        this.onResolvedCallbacks.push(() => handle(onFulfilled, this.value));
        this.onRejectedCallbacks.push(() => handle(onRejected, this.reason));
      }
    });
  }
}

console.log('1. 开始测试');

const p1 = new MyPromise((resolve, reject) => {
  console.log('2. 进入 Promise 执行器');
  // 模拟异步操作，比如接口请求
  setTimeout(() => {
    resolve('3. 异步操作成功！');
  }, 1000);
});

p1.then((res) => {
  console.log('4. 接收到结果:', res);
});

console.log('5. 同步代码执行完毕');

/*
预期打印顺序：
1. 开始测试
2. 进入 Promise 执行器
3. 同步代码执行完毕
(等待 1 秒后)
4. 接收到结果: 3. 异步操作成功！
*/
```

### 2. 手写promise.all

**核心逻辑：** 接收一个 Promise 数组，只有当所有 Promise 都成功时才返回结果数组；只要有一个失败，就直接 Reject。

```JavaScript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError("Arguments must be an array"));
    }

    let result = [];
    let count = 0;
    promises.forEach((p, index) => {
      // 包装成 Promise.resolve 以兼容非 Promise 对象
      Promise.resolve(p).then(
        (res) => {
          result[index] = res; // 保证结果顺序
          count++;
          if (count === promises.length) resolve(result);
        },
        (err) => reject(err)
      );
    });
  });
}
```

- **讲解：** 关键点在于 `result[index]` 必须根据索引赋值，而不是简单的 `push`，这样才能保证输出顺序与输入顺序一致。

## 4. 手写 new 操作符

**核心逻辑：** 模拟构造函数的实例化过程。

```JavaScript
function myNew(constructor, ...args) {
  // 1. 创建一个空对象，并将其原型指向构造函数的 prototype
  const obj = Object.create(constructor.prototype);
  // 2. 执行构造函数，并将 this 绑定到这个新对象上
  const res = constructor.apply(obj, args);
  // 3. 如果构造函数返回的是对象，则返回该对象；否则返回新创建的 obj
  return res instanceof Object ? res : obj;
}
```

## 5. 数组扁平化 (Array Flatten)

**题目：** 将 `[1, [2, [3]], 4]` 变成 `[1, 2, 3, 4]`。

### 方案 A：递归法 (最通用)

```JavaScript
function flatten(arr) {
  let result = [];
  for (let i = 0; i < arr.length; i++) {
    if (Array.isArray(arr[i])) {
      result = result.concat(flatten(arr[i]));
    } else {
      result.push(arr[i]);
    }
  }
  return result;
}
```

### 方案 B：ES6 的 flat 方法

```JavaScript
// 简单面试可以直接写这个，但面试官通常想看上面的递归实现
arr.flat(Infinity);
```

## 6. 发布订阅模式 (EventEmitter)

**核心逻辑：** 这是 Vue 和 Node.js 事件机制的核心。你需要维护一个“调度中心”（通常是一个对象），通过 `on` 存入回调，`emit` 触发回调。

```JavaScript
class EventEmitter {
  constructor() {
    this.events = {}; // 存储事件名和对应的回调数组
  }

  on(eventName, cb) {
    if (!this.events[eventName]) {
      this.events[eventName] = [];
    }
    this.events[eventName].push(cb);
  }

  emit(eventName, ...args) {
    if (this.events[eventName]) {
      this.events[eventName].forEach(cb => cb.apply(this, args));
    }
  }

  off(eventName, cb) {
    if (this.events[eventName]) {
      this.events[eventName] = this.events[eventName].filter(fn => fn !== cb);
    }
  }

  once(eventName, cb) {
    // 包装一层：执行后立即销毁
    const oneTime = (...args) => {
      cb.apply(this, args);
      this.off(eventName, oneTime);
    };
    this.on(eventName, oneTime);
  }
}
```

- **讲解：** `once` 的实现是难点，关键在于创建一个临时的包装函数，它在执行完业务逻辑后会自动把自己“踢出”订阅列表。

## 7. 函数柯里化 (Currying)

**核心逻辑：** 将一个接收多个参数的函数，转化为接收单一参数并返回新函数的形式。利用闭包保存已传入的参数。

```JavaScript
function curry(fn) {
  return function curried(...args) {
    // 如果传入参数足够了，直接执行原函数
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    } else {
      // 否则返回一个新函数，继续接收剩下的参数
      return function(...args2) {
        return curried.apply(this, args.concat(args2));
      };
    }
  };
}

// 测试
const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);
console.log(curriedAdd(1)(2)(3)); // 6
```

- **讲解：** `fn.length` 可以获取函数定义时的形参个数。这是递归判断是否该终止并执行的关键。

## 8. 手写数组去重 (Unique)

面试官通常会让你给出多种方案，从简单到复杂。

### 方案 A：Set (最快)

```JavaScript
const unique = (arr) => [...new Set(arr)];
```

### 方案 B：filter + indexOf (最经典)

```JavaScript
function unique(arr) {
  return arr.filter((item, index) => {
    // 只有第一个出现的元素，其索引才会等于当前的 index
    return arr.indexOf(item) === index;
  });
}
```

## 9. 手写 call、apply、bind

**核心逻辑：** 改变 `this` 指向。原理是把函数挂载到目标对象上作为其属性执行，执行后再删掉该属性。
### 1. 手写 `call`

`call` 的特点是参数平铺（即 `arg1, arg2...`）。

```JavaScript
Function.prototype.myCall = function(context, ...args) {
  // 1. 处理 context 为 null 或 undefined 的情况，默认指向 window (或 globalThis)
  context = context || window;

  // 2. 使用 Symbol 创建唯一的 key，防止覆盖对象原有的属性
  const fnKey = Symbol();

  // 3. 将当前函数（this）赋值给 context 的一个属性
  // 这里的 this 指向的就是调用 myCall 的那个函数
  context[fnKey] = this;

  // 4. 执行函数并保存结果
  const result = context[fnKey](...args);

  // 5. 擦除痕迹：删除临时属性
  delete context[fnKey];

  // 6. 返回执行结果
  return result;
};
```

### 2. 手写 `apply`

`apply` 与 `call` 的唯一区别在于它接收的是**一个数组**作为参数。

```JavaScript
Function.prototype.myApply = function(context, argsArray) {
  context = context || window;
  const fnKey = Symbol();
  context[fnKey] = this;

  let result;
  // 判断是否有参数传入，apply 的第二个参数必须是数组或类数组
  if (!Array.isArray(argsArray)) {
    result = context[fnKey]();
  } else {
    result = context[fnKey](...argsArray);
  }

  delete context[fnKey];
  return result;
};
```

### 3. 手写 `bind` (最难的一道)

`bind` 不会立即执行函数，而是返回一个**新函数**。

> **注意点：** > 1. 需要支持“柯里化”（预设部分参数）。 2. **核心难点：** 如果返回的函数被当做 `new` 构造函数使用，原本绑定的 `this` 应该失效。

```JavaScript
Function.prototype.myBind = function(context, ...args) {
  // 保存当前函数
  const self = this;

  return function F(...newArgs) {
    // 考虑 bind 返回的函数作为构造函数被 new 的情况
    // 如果 this 是 F 的实例，说明正在执行 new F()
    if (this instanceof F) {
      return new self(...args, ...newArgs);
    }
    
    // 正常调用，合并两次传入的参数
    return self.apply(context, args.concat(newArgs));
  };
};
```

## 10. 异步串行执行 (Array Reduce 妙用)

**场景：** 有多个异步任务，需要等第一个完成后再执行第二个（类似中间件流）。

```JavaScript
function runPromisesInSeries(promises) {
  return promises.reduce((promiseChain, currentTask) => {
    return promiseChain.then(chainResults => {
      return currentTask().then(currentResult => {
        return [...chainResults, currentResult];
      });
    });
  }, Promise.resolve([]));
}
```

- **讲解：** 利用 `reduce` 初始值传入一个 `Promise.resolve([])`，每一轮迭代都在前一个 Promise 的 `.then` 后面挂载新的 Promise。

## 11. 版本号排序

版本号排序（Version Sorting）是前端面试中非常经典的一道题。它不仅考察你对数组 `sort` 方法的理解，更考察你对**字符串处理**、**补齐思想**以及**边界情况**的思考。

常见的版本号格式如：`1.2.1`, `1.0.0`, `1.10.2` 等。

版本号排序不能直接用字符串的 `localeCompare`，因为字符串比较是逐字符的。

> **错误示例：** `"1.10.2" < "1.2.1"` 在字符串比较中是成立的（因为 '1' 对 '1'，'.' 对 '.'，然后 '1' 小于 '2'）。但在版本号逻辑中，`1.10` 显然比 `1.2` 大。

**解题思路：**

1. 将版本号字符串通过 `.` 分隔成数组。
    
2. 比较数组对应位置的数字。
    
3. 如果长度不一致，短的版本号需要向长的看齐（补 0）。
    
### 方案A：标准循环对比法

这是最稳健、可读性最好的实现方式。

```JavaScript
function compareVersion(versions) {
  return versions.sort((a, b) => {
    const arr1 = a.split('.');
    const arr2 = b.split('.');
    
    // 取最长的版本号长度作为循环终点
    const maxLength = Math.max(arr1.length, arr2.length);

    for (let i = 0; i < maxLength; i++) {
      // 如果该位不存在，则默认为 0
      const num1 = Number(arr1[i] || 0);
      const num2 = Number(arr2[i] || 0);

      if (num1 > num2) return 1;
      if (num1 < num2) return -1;
      // 如果相等，则继续比较下一位
    }
    return 0;
  });
}

// 测试
const versions = ['1.2.1', '1.0.2', '1.10.2', '1.1', '2.0'];
console.log(compareVersion(versions)); 
// 输出: ["1.0.2", "1.1", "1.2.1", "1.10.2", "2.0"]
```

### 方案B：利用填充（Padding）

如果你想追求代码简洁，可以将所有版本号通过 `padStart` 补齐到相同的位数，然后直接进行字符串比较。

```JavaScript
function sortVersions(versions) {
  return versions.sort((a, b) => {
    const aParts = a.split('.');
    const bParts = b.split('.');
    const len = Math.max(aParts.length, bParts.length);

    // 将每一部分都补齐到相同长度，例如 1.2 变成 0001.0002
    const format = (v) => v.split('.')
                           .map(num => num.padStart(10, '0'))
                           .join('.');

    return format(a).localeCompare(format(b));
  });
}
```

## 12. 扁平化和反扁平

### 扁平化

```js
function flatten(arr) {
	while(arr.some((item) => Array.isArray(item))){
		arr = [].concat(...arr)
	}
}
const arr = [1, [2, [3, 4]]]
```

### 反扁平

```js
// 后端返回的扁平数组（带 id 和 parentId）
const flatList = [
  { id: 1, name: '一级菜单1', parentId: null },
  { id: 2, name: '二级菜单1-1', parentId: 1 },
  { id: 3, name: '三级菜单1-1-1', parentId: 2 },
  { id: 4, name: '二级菜单1-2', parentId: 1 },
  { id: 5, name: '一级菜单2', parentId: null }
];

function listToTree(list) {
	const map = {}
	const tree = []
	list.foreach((item) => {
		map[item.id] = {...item, children: []}
	})
	
	list.foreach((item) => {
		const node = map[item.id]
		if(item.parentId === null){
			tree.push(node)
		} else {
			const parent = map[item.parentId]
			if(parent){
				parent.children.push(node)
			}
		}
	})
	return tree
}

// 测试
const tree = listToTree(flatList);
console.log(tree);
/* 输出：
[
  {
    id: 1,
    name: '一级菜单1',
    parentId: null,
    children: [
      {
        id: 2,
        name: '二级菜单1-1',
        parentId: 1,
        children: [
          { id: 3, name: '三级菜单1-1-1', parentId: 2, children: [] }
        ]
      },
      { id: 4, name: '二级菜单1-2', parentId: 1, children: [] }
    ]
  },
  { id: 5, name: '一级菜单2', parentId: null, children: [] }
]
*/
```