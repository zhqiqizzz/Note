Promise 是 ES6 引入的**异步编程解决方案**，核心作用是解决传统回调函数导致的 “回调地狱” 问题，让异步代码的流程更清晰、更易维护，是目前前端处理异步操作（如接口请求、文件读取、定时器）的最主流方式。

## 一、Promise 是什么

Promise 是一个**表示异步操作最终状态（成功或失败）的对象**，它像一个 “容器”，里面装着一个未来才会完成的异步操作。通过 Promise，你可以在异步操作完成后，用更优雅的方式处理结果（成功）或错误（失败）。

### 解决的痛点：回调地狱

在没有 Promise 之前，处理多个串行异步操作只能嵌套回调函数，代码会变得难以阅读和维护，这就是 “回调地狱”：

```javascript
// 传统回调地狱示例：串行请求3个接口
ajax('/api/user', (user) => {
  ajax(`/api/orders/${user.id}`, (orders) => {
    ajax(`/api/details/${orders[0].id}`, (details) => {
      console.log(details);
      // 嵌套越来越深，代码难以维护
    }, (err) => console.error(err));
  }, (err) => console.error(err));
}, (err) => console.error(err));
```

用 Promise 后，代码可以写成链式调用，清晰很多：

```javascript
// Promise 链式调用示例
fetch('/api/user')
  .then(user => fetch(`/api/orders/${user.id}`))
  .then(orders => fetch(`/api/details/${orders[0].id}`))
  .then(details => console.log(details))
  .catch(err => console.error(err));
```

## 二、Promise 的三种状态

Promise 有且只有三种状态，**状态一旦改变就不可逆**（只能从 pending 变为 fulfilled 或 rejected，变了之后就固定了）：

1. **pending（进行中）**：初始状态，异步操作还没完成；
2. **fulfilled（已成功）**：异步操作成功完成，此时会调用 `.then()` 的第一个回调；
3. **rejected（已失败）**：异步操作失败，此时会调用 `.then()` 的第二个回调或 `.catch()`。

## 三、Promise 的基本用法

### 1. 创建 Promise

用 `new Promise()` 构造函数创建，接收一个 **执行器函数（executor）** 作为参数，执行器函数会立即同步执行，里面放异步操作。执行器函数有两个参数：

- `resolve`：调用后将 Promise 状态改为 `fulfilled`，并传递成功结果；
- `reject`：调用后将 Promise 状态改为 `rejected`，并传递失败原因。

```javascript
// 示例：用 Promise 封装一个定时器
const delay = (ms) => {
  return new Promise((resolve, reject) => {
    // 异步操作：定时器
    setTimeout(() => {
      if (ms > 0) {
        resolve(`延迟了 ${ms} 毫秒`); // 成功，调用 resolve
      } else {
        reject(new Error('延迟时间必须大于0')); // 失败，调用 reject
      }
    }, ms);
  });
};
```

### 2. 处理结果：.then () 和 .catch ()

- **`.then(onFulfilled, onRejected)`**：接收两个可选回调，第一个处理成功（fulfilled），第二个处理失败（rejected）；
- **`.catch(onRejected)`**：专门处理失败（rejected），相当于 `.then(undefined, onRejected)`，更常用。

```javascript
// 使用上面的 delay Promise
delay(1000)
  .then(result => {
    console.log(result); // 1秒后输出：延迟了 1000 毫秒
    return delay(500); // .then() 返回新的 Promise，支持链式调用
  })
  .then(result => {
    console.log(result); // 再等0.5秒输出：延迟了 500 毫秒
  })
  .catch(err => {
    console.error(err.message); // 任何一个环节失败都会到这里
  });
```

### 3. 链式调用的核心

`.then()` 和 `.catch()` 都会**返回一个新的 Promise**，所以可以一直链式调用下去。如果 `.then()` 的回调里返回一个值，这个值会被包装成 `fulfilled` 的 Promise；如果抛出错误，会被包装成 `rejected` 的 Promise。

## 四、Promise 的静态方法（前端最常用）

这些是 `Promise` 构造函数上的方法，不用 `new`，直接调用，用于处理多个 Promise 的组合场景。

### 1. Promise.all ()：并发请求，全部成功才成功

**作用**：接收一个 Promise 数组，返回一个新 Promise。只有当数组里**所有 Promise 都成功**时，新 Promise 才成功（返回所有成功结果的数组）；只要有**一个 Promise 失败**，新 Promise 就立即失败（返回第一个失败的原因）。

**适用场景**：并发请求多个接口，等所有接口都返回后再处理数据（比如页面初始化时同时请求用户信息、商品列表、订单列表）。

```javascript
// 示例：并发请求3个接口
const fetchUser = fetch('/api/user');
const fetchOrders = fetch('/api/orders');
const fetchDetails = fetch('/api/details');

Promise.all([fetchUser, fetchOrders, fetchDetails])
  .then(([user, orders, details]) => {
    // 所有接口都成功，按顺序返回结果
    console.log('用户信息:', user);
    console.log('订单列表:', orders);
    console.log('详情:', details);
  })
  .catch(err => {
    // 任何一个接口失败，立即到这里
    console.error('有接口失败:', err);
  });
```

### 2. Promise.race ()：竞速，最快的那个决定状态

**作用**：接收一个 Promise 数组，返回一个新 Promise。数组里**最快改变状态的 Promise**（不管成功还是失败），它的状态和结果就决定了新 Promise 的状态和结果。

**适用场景**：接口超时控制（比如给请求加一个超时定时器，超过时间就返回失败）。

```javascript
// 示例：接口超时控制
const fetchWithTimeout = (url, timeoutMs) => {
  // 接口请求 Promise
  const fetchPromise = fetch(url);
  // 超时 Promise：超过时间就 reject
  const timeoutPromise = new Promise((_, reject) => {
    setTimeout(() => reject(new Error('请求超时')), timeoutMs);
  });
  // 竞速：哪个先完成用哪个
  return Promise.race([fetchPromise, timeoutPromise]);
};

// 使用：3秒超时
fetchWithTimeout('/api/user', 3000)
  .then(user => console.log(user))
  .catch(err => console.error(err.message)); // 超过3秒输出“请求超时”
```

### 3. Promise.allSettled ()：等待所有 Promise 完成，不管成功失败

**作用**：接收一个 Promise 数组，返回一个新 Promise。等数组里**所有 Promise 都完成**（不管成功还是失败）后，新 Promise 才成功，返回一个包含每个 Promise 结果的对象数组（每个对象有 `status`：`fulfilled`/`rejected`，`value`/`reason`）。

**适用场景**：并发请求多个接口，即使部分失败也要处理成功的结果（比如上传多张图片，部分失败也要记录成功的图片 URL）。

```javascript
// 示例：上传多张图片，部分失败也处理
const uploadImg1 = fetch('/api/upload', { method: 'POST', body: img1 });
const uploadImg2 = fetch('/api/upload', { method: 'POST', body: img2 });
const uploadImg3 = fetch('/api/upload', { method: 'POST', body: img3 });

Promise.allSettled([uploadImg1, uploadImg2, uploadImg3])
  .then(results => {
    // results 是数组，每个元素对应一个 Promise 的结果
    const successUrls = results
      .filter(res => res.status === 'fulfilled')
      .map(res => res.value.url);
    const failCount = results.filter(res => res.status === 'rejected').length;
    console.log('成功的图片URL:', successUrls);
    console.log('失败数量:', failCount);
  });
```

### 4. Promise.any ()：只要有一个成功就成功，全部失败才失败

**作用**：接收一个 Promise 数组，返回一个新 Promise。只要数组里**有一个 Promise 成功**，新 Promise 就立即成功（返回第一个成功的结果）；只有当**所有 Promise 都失败**时，新 Promise 才失败（返回一个 AggregateError，包含所有失败原因）。

**适用场景**：从多个备用接口中选最快成功的那个（比如有多个 CDN 接口，选最快返回的）。

### 5. Promise.resolve () / Promise.reject ()：快速创建已完成的 Promise

- `Promise.resolve(value)`：快速创建一个 `fulfilled` 的 Promise，直接返回成功结果；
- `Promise.reject(reason)`：快速创建一个 `rejected` 的 Promise，直接返回失败原因。

```javascript
Promise.resolve('成功了').then(res => console.log(res)); // 输出：成功了
Promise.reject(new Error('失败了')).catch(err => console.error(err.message)); // 输出：失败了
```

## 五、async/await：Promise 的语法糖

ES2017 引入的 `async/await` 是 Promise 的**语法糖**，让异步代码看起来像同步代码一样，更易读易维护，是目前前端写异步的主流方式。

### 1. 基本用法

- **`async` 函数**：用 `async` 关键字声明的函数，返回值一定是 Promise（即使返回普通值，也会被包装成 Promise）；
- **`await` 关键字**：只能在 `async` 函数里用，后面跟一个 Promise，会**暂停 async 函数的执行**，直到 Promise 状态改变，然后返回结果。

```javascript
// 示例：用 async/await 重写串行请求
async function getDetails() {
  try {
    // 等待第一个接口返回，再执行下一行
    const user = await fetch('/api/user');
    // 等待第二个接口返回
    const orders = await fetch(`/api/orders/${user.id}`);
    // 等待第三个接口返回
    const details = await fetch(`/api/details/${orders[0].id}`);
    console.log(details);
  } catch (err) {
    // 用 try/catch 捕获错误，替代 .catch()
    console.error(err);
  }
}

// 调用 async 函数
getDetails();
```

### 2. async/await 处理并发

如果用 `await` 串行处理多个 Promise 会慢，可以结合 `Promise.all()` 实现并发：

```javascript
async function getAllData() {
  try {
    // 并发请求3个接口，同时开始，等所有返回
    const [user, orders, details] = await Promise.all([
      fetch('/api/user'),
      fetch('/api/orders'),
      fetch('/api/details')
    ]);
    console.log(user, orders, details);
  } catch (err) {
    console.error(err);
  }
}
```

## 六、前端常见应用场景

### 1. 封装接口请求

用 Promise 封装 `fetch` 或 `XMLHttpRequest`，统一处理成功和失败：

```javascript
// 封装 fetch 请求
const request = (url, options = {}) => {
  return new Promise((resolve, reject) => {
    fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      }
    })
      .then(res => {
        if (!res.ok) {
          reject(new Error(`请求失败，状态码：${res.status}`));
        }
        return res.json();
      })
      .then(data => resolve(data))
      .catch(err => reject(err));
  });
};

// 使用
request('/api/user')
  .then(user => console.log(user))
  .catch(err => console.error(err));
```

### 2. 串行执行异步任务

比如依次上传多张图片，上一张上传成功再传下一张：

```javascript
// 图片数组
const images = [img1, img2, img3];

// 串行上传
async function uploadImagesSequentially() {
  const results = [];
  for (const img of images) {
    // 等待当前图片上传成功，再传下一张
    const res = await request('/api/upload', { method: 'POST', body: img });
    results.push(res.url);
  }
  return results;
}
```

## 七、常见避坑点

1. **Promise 状态不可逆**：一旦调用 `resolve` 或 `reject`，状态就固定了，再调用不会有任何效果；
2. **.then () 里的错误要捕获**：如果 `.then()` 的回调里抛出错误，没有 `.catch()` 或第二个回调的话，错误会被 “吞掉”，不会在控制台显示；
3. **async 函数里的错误要用 try/catch**：`await` 后面的 Promise 失败会抛出错误，必须用 `try/catch` 捕获，否则会导致程序崩溃；
4. **Promise.all () 是 “失败快速返回”**：如果需要处理部分成功的结果，用 `Promise.allSettled()` 代替。

## 总结

- Promise 是处理异步操作的核心，解决回调地狱问题；
- 三种状态：pending、fulfilled、rejected，状态不可逆；
- 常用静态方法：`Promise.all()`（并发全成功）、`Promise.race()`（竞速）、`Promise.allSettled()`（等待所有完成）；
- `async/await` 是语法糖，让异步代码更易读，是目前主流写法；
- 注意错误捕获和状态不可逆的坑。