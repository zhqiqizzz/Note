Web Worker 是 HTML5 引入的关键技术，它允许 JavaScript 在**后台线程**中运行，解决了 JavaScript 单线程模型的性能瓶颈问题。下面我将从多个维度详细介绍 Web Worker。
## 一、什么是 Web Worker？

### 1.1 为什么需要 Web Worker

JavaScript 是**单线程语言**，同一时间只能执行一个任务。当执行耗时操作（如大量数据计算、复杂循环）时，会阻塞主线程，导致：

- 页面"卡死"或"无响应"
- UI 渲染中断
- 用户交互延迟

Web Worker 通过创建**独立的工作线程**，将耗时任务移到后台执行，主线程可以继续响应用户操作，保持页面流畅。
### 1.2 核心特性

|特性|说明|
|---|---|
|**多线程**|在后台独立线程运行，不阻塞主线程|
|**消息通信**|通过 `postMessage` 和 `onmessage` 与主线程通信|
|**独立作用域**|拥有独立的全局上下文（`self` 而非 `window`）|
|**不能操作 DOM**|Worker 线程无法直接访问或修改 DOM|
|**同源策略**|Worker 脚本必须与主页面同源|

## 二、Web Worker 的类型

Web Worker 主要有三种类型：

### 2.1 专用 Worker（Dedicated Worker）

- **特点**：每个页面独占一个 Worker 实例
- **生命周期**：随页面关闭而销毁
- **适用场景**：单个页面的耗时计算任务

```javascript
// 主线程创建 Worker
const worker = new Worker('worker.js');

// 发送数据
worker.postMessage({ data: 'Hello Worker' });

// 接收消息
worker.onmessage = function(e) {
    console.log('收到:', e.data);
};

// 终止 Worker
worker.terminate();
```

### 2.2 共享 Worker（Shared Worker）

- **特点**：多个页面/标签页可共享同一个 Worker 实例
- **生命周期**：只要有一个页面连接，Worker 就持续存在
- **适用场景**：跨页面通信、共享状态（如聊天室、多标签页数据同步）

```javascript
// 创建 Shared Worker
const sharedWorker = new SharedWorker('shared-worker.js');

// 必须启动端口
sharedWorker.port.start();

// 发送消息
sharedWorker.port.postMessage('Hello');

// 接收消息
sharedWorker.port.onmessage = function(e) {
    console.log('收到:', e.data);
};
```

### 2.3 Service Worker

- **特点**：独立于页面运行，可拦截网络请求
- **生命周期**：独立于页面，可长期存活
- **适用场景**：离线缓存、推送通知、PWA 应用

```javascript
// 注册 Service Worker
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
}
```

### 2.4 三种 Worker 对比

| 类型               | 作用域    | 生命周期      | 主要用途    |
| ---------------- | ------ | --------- | ------- |
| Dedicated Worker | 单个页面   | 随页面关闭     | 耗时计算    |
| Shared Worker    | 多个页面共享 | 所有连接关闭后销毁 | 跨页面通信   |
| Service Worker   | 整个网站   | 独立持久      | 离线缓存、推送 |

## 三、基本使用方法

### 3.1 创建 Worker 文件（worker.js）

```javascript
// worker.js
self.onmessage = function(e) {
    const data = e.data;
    
    // 执行耗时计算
    const result = heavyComputation(data);
    
    // 发送结果回主线程
    self.postMessage(result);
};

function heavyComputation(input) {
    // 模拟耗时操作
    let sum = 0;
    for (let i = 0; i < 1000000; i++) {
        sum += i;
    }
    return sum;
}
```

### 3.2 主线程使用

```javascript
// main.js
const worker = new Worker('worker.js');

// 发送数据
worker.postMessage({ value: 100 });

// 接收结果
worker.onmessage = function(e) {
    console.log('计算结果:', e.data);
};

// 错误处理
worker.onerror = function(error) {
    console.error('Worker 错误:', error);
};
```

## 四、通信机制

### 4.1 消息传递

Worker 与主线程通过**消息传递**通信，数据会被**结构化克隆**（复制），而非共享引用。

```javascript
// 主线程 → Worker
worker.postMessage({
    type: 'CALCULATE',
    payload: { numbers: [1, 2, 3, 4, 5] }
});

// Worker → 主线程
self.postMessage({
    type: 'RESULT',
    payload: { sum: 15 }
});
```

### 4.2 传递大数据（Transferable Objects）

对于大数据（如 ArrayBuffer），可使用 `transfer` 选项实现**零拷贝**传输：

```javascript
// 主线程
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
worker.postMessage(buffer, [buffer]); // 所有权转移

// Worker
self.onmessage = function(e) {
    const buffer = e.data; // 直接获取，无复制
};
```

## 五、应用场景

### 5.1 CPU 密集型计算

- 大量数据排序/过滤
- 复杂数学运算
- 加密/解密

### 5.2 图像/视频处理

- 图片压缩、滤镜处理
- 视频帧分析
- Canvas 渲染

### 5.3 实时数据流处理

- 高频数据更新
- 实时图表绘制
- 日志分析

### 5.4 其他场景

- WebAssembly 加速计算
- 游戏物理模拟
- AI 推理（前端机器学习）
- 后台文件处理

---

## 六、限制与注意事项

| 限制           | 说明                              |
| ------------ | ------------------------------- |
| ❌ 不能操作 DOM   | Worker 无 `document`、`window` 对象 |
| ❌ 不能访问部分 API | 如 `localStorage`、`alert` 等      |
| ✅ 可使用网络 API  | `fetch`、`XMLHttpRequest`        |
| ✅ 可使用定时器     | `setTimeout`、`setInterval`      |
| ⚠️ 同源策略      | Worker 脚本必须同源                   |
| ⚠️ 资源开销      | 每个 Worker 消耗独立内存和 CPU           |

### 6.1 可用全局对象

Worker 中可用的全局对象包括：

- `self`（Worker 全局作用域）
- `navigator`
- `location`（只读）
- `importScripts()`（导入其他脚本）

---

## 七、Vue3/React 中的最佳实践

### 7.1 Vue3 封装 Hook

```javascript
// useWorker.js
import { ref, onUnmounted } from 'vue';

export function useWorker(workerPath) {
    const worker = ref(null);
    const result = ref(null);
    
    const initWorker = () => {
        worker.value = new Worker(workerPath);
        worker.value.onmessage = (e) => {
            result.value = e.data;
        };
    };
    
    const postMessage = (data) => {
        worker.value?.postMessage(data);
    };
    
    const terminate = () => {
        worker.value?.terminate();
    };
    
    onUnmounted(() => terminate());
    
    return { initWorker, postMessage, result, terminate };
}
```

### 7.2 使用 Worker Pool（任务池）

对于频繁创建/销毁 Worker 的场景，可使用 Worker 池复用：

```javascript
class WorkerPool {
    constructor(workerPath, poolSize = 4) {
        this.workers = [];
        for (let i = 0; i < poolSize; i++) {
            this.workers.push(new Worker(workerPath));
        }
    }
    
    runTask(data) {
        const worker = this.workers.find(w => !w.busy);
        if (worker) {
            worker.busy = true;
            return new Promise((resolve) => {
                worker.onmessage = (e) => {
                    worker.busy = false;
                    resolve(e.data);
                };
                worker.postMessage(data);
            });
        }
    }
}
```

## 八、高级特性

### 8.1 SharedArrayBuffer + Atomics

实现真正的**共享内存**和线程同步（需跨域隔离头支持）：

```javascript
// 创建共享内存
const sharedBuffer = new SharedArrayBuffer(1024);
const sharedArray = new Int32Array(sharedBuffer);

// Worker 中使用 Atomics 进行原子操作
Atomics.add(sharedArray, 0, 1);
Atomics.wait(sharedArray, 0, expectedValue);
```

### 8.2 模块化工人（Module Workers）


```javascript
// 使用 ES Module 创建 Worker
const worker = new Worker('worker.js', { type: 'module' });

// worker.js 中可使用 import/export
import { helper } from './utils.js';
export const VERSION = '1.0';
```

## 九、总结

|优势|劣势|
|---|---|
|✅ 不阻塞 UI 主线程|❌ 不能操作 DOM|
|✅ 充分利用多核 CPU|❌ 通信有序列化开销|
|✅ 隔离高风险任务|❌ 增加代码复杂度|
|✅ 支持多种 Worker 类型|❌ 浏览器兼容性需注意|

**最佳实践建议**：

1. 仅对**真正耗时**的任务使用 Worker
2. 避免频繁创建/销毁 Worker，使用 Worker 池
3. 大数据传输使用 `transfer` 选项
4. 做好错误处理和 Worker 终止
5. 考虑使用模块化工人提升代码可维护性