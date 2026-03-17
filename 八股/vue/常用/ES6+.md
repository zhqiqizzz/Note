ES6+ 是对 **ECMAScript 2015（ES6）及后续版本**的统称，它是 JavaScript 的语言标准，通过一系列新特性让代码更简洁、更强大、更易维护。以下是 ES6+ 最核心、最常用的特性详解：
## 一、变量声明：`let` / `const`

### 1. `let`：块级作用域变量

- **特点**：
    
    - 块级作用域（`{}` 内有效）。
    - 不存在变量提升（不能在声明前使用）。
    - 同一作用域内不能重复声明。
- **示例**：
    
```javascript
    if (true) {
      let a = 10;
      console.log(a); // 10
    }
    console.log(a); // 报错：a is not defined
```

### 2. `const`：常量声明

- **特点**：
    
    - 声明时必须赋值，且赋值后**不能重新赋值**。
    - 若赋值为对象 / 数组，**其内部属性 / 元素可以修改**（因为 `const` 只保证引用地址不变）。
    
- **示例**：
    
    j
```javascript
    const PI = 3.14;
    PI = 3.1415; // 报错：Assignment to constant variable
    
    const obj = { name: '张三' };
    obj.name = '李四'; // ✅ 允许：修改对象内部属性
```
    

### 3. 对比 `var`

表格

|特性|`var`|`let`/`const`|
|---|---|---|
|作用域|函数作用域|块级作用域|
|变量提升|有（声明前可访问）|无（声明前访问报错）|
|重复声明|允许|不允许|
|重新赋值|允许|`let` 允许，`const` 不允许|

## 二、解构赋值

### 1. 数组解构

从数组中提取值，按位置对应赋值给变量。

javascript

运行

```
const [a, b, c] = [1, 2, 3];
console.log(a); // 1
console.log(b); // 2

// 跳过元素
const [, , third] = [1, 2, 3];
console.log(third); // 3

// 默认值
const [x = 10, y = 20] = [5];
console.log(x); // 5
console.log(y); // 20
```

### 2. 对象解构

从对象中提取值，按属性名对应赋值给变量。

javascript

运行

```
const user = { name: '张三', age: 25 };
const { name, age } = user;
console.log(name); // 张三
console.log(age); // 25

// 重命名变量
const { name: username, age: userAge } = user;
console.log(username); // 张三

// 默认值
const { gender = '男' } = user;
console.log(gender); // 男
```

### 3. 函数参数解构

javascript

运行

```
// 直接在参数中解构对象
function greet({ name, age = 18 }) {
  console.log(`你好，${name}，今年${age}岁`);
}

const user = { name: '李四' };
greet(user); // 你好，李四，今年18岁
```

## 三、函数扩展

### 1. 箭头函数

- **特点**：
    
    - 语法更简洁（省略 `function` 关键字）。
    - **没有自己的 `this`**，继承外层作用域的 `this`（解决普通函数 `this` 指向混乱的问题）。
    - 不能作为构造函数（不能用 `new`）。
    
- **示例**：
    
    javascript
    
    运行
    
    ```
    // 普通函数
    const add = function(a, b) {
      return a + b;
    };
    
    // 箭头函数（简写）
    const addArrow = (a, b) => a + b;
    
    // 单参数可省略括号
    const double = n => n * 2;
    
    // 箭头函数的 this 继承外层
    const obj = {
      name: '张三',
      sayHi() {
        setTimeout(() => {
          console.log(`你好，${this.name}`); // this 指向 obj
        }, 1000);
      }
    };
    obj.sayHi(); // 你好，张三
    ```
    

### 2. 默认参数

javascript

运行

```
function greet(name = '陌生人') {
  console.log(`你好，${name}`);
}
greet(); // 你好，陌生人
greet('张三'); // 你好，张三
```

### 3. 剩余参数（`...args`）

将剩余的参数收集为一个数组。

javascript

运行

```
function sum(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3)); // 6
```

### 4. 展开运算符（`...`）

将数组 / 对象 “展开” 为独立的元素 / 属性。

javascript

运行

```
// 数组展开
const arr1 = [1, 2];
const arr2 = [...arr1, 3, 4]; // [1,2,3,4]

// 对象展开
const obj1 = { name: '张三' };
const obj2 = { ...obj1, age: 25 }; // { name: '张三', age: 25 }

// 函数调用时展开数组
const nums = [1, 2, 3];
console.log(Math.max(...nums)); // 3
```

## 四、对象扩展

### 1. 对象字面量简写

javascript

运行

```
const name = '张三';
const age = 25;

// 传统写法
const user1 = { name: name, age: age };

// 简写：属性名与变量名相同时可省略
const user2 = { name, age };

// 方法简写
const obj = {
  sayHi() {
    console.log('Hi');
  }
};
```

### 2. 可选链操作符（`?.`）

安全访问对象的深层属性，避免 `Cannot read property 'xxx' of undefined` 报错。

javascript

运行

```
const user = { address: { city: '北京' } };

// 传统写法（层层判断）
const city1 = user && user.address && user.address.city;

// 可选链写法
const city2 = user?.address?.city; // 北京
const street = user?.address?.street; // undefined（不报错）
```

### 3. 空值合并操作符（`??`）

为 `null`/`undefined` 提供默认值（区别于 `||`，`||` 会把 `0`/`''`/`false` 也当作假值）。

javascript

运行

```
const count = 0;
const defaultCount1 = count || 10; // 10（|| 把 0 当作假值）
const defaultCount2 = count ?? 10; // 0（?? 只对 null/undefined 生效）
```

### 4. `Object.keys` / `Object.values` / `Object.entries`

javascript

运行

```
const user = { name: '张三', age: 25 };

Object.keys(user); // ['name', 'age']（获取键数组）
Object.values(user); // ['张三', 25]（获取值数组）
Object.entries(user); // [['name', '张三'], ['age', 25]]（获取键值对数组）
```

## 五、数组扩展

### 1. `Array.from`：将类数组 / 可迭代对象转为数组

javascript

运行

```
// 类数组（如 arguments、DOM 节点列表）
function fn() {
  const args = Array.from(arguments);
  console.log(args); // [1, 2, 3]
}
fn(1, 2, 3);

// 可迭代对象（如 Set、Map）
const set = new Set([1, 2, 3]);
const arr = Array.from(set); // [1, 2, 3]
```

### 2. `find` / `findIndex`：查找数组元素

javascript

运行

```
const users = [
  { id: 1, name: '张三' },
  { id: 2, name: '李四' }
];

// find：找到第一个符合条件的元素
const user = users.find(u => u.id === 2);
console.log(user); // { id: 2, name: '李四' }

// findIndex：找到第一个符合条件的元素索引
const index = users.findIndex(u => u.id === 2);
console.log(index); // 1
```

### 3. `includes`：判断数组是否包含某元素

javascript

运行

```
const arr = [1, 2, 3];
arr.includes(2); // true
arr.includes(4); // false
```

### 4. `flat` / `flatMap`：数组扁平化

javascript

运行

```
// flat：将多维数组转为一维数组（默认扁平化1层，可传数字指定层数）
const arr = [1, [2, [3, 4]]];
arr.flat(); // [1, 2, [3, 4]]
arr.flat(2); // [1, 2, 3, 4]

// flatMap：先 map 再 flat（扁平化1层）
const arr2 = ['hello world', 'foo bar'];
arr2.flatMap(s => s.split(' ')); // ['hello', 'world', 'foo', 'bar']
```

## 六、异步编程：`Promise` + `async/await`

### 1. `Promise`（ES6）

用于解决 “回调地狱” 问题，代表一个异步操作的最终状态（成功 / 失败）。

- **状态**：`pending`（进行中）→ `fulfilled`（成功）/ `rejected`（失败）。
- **基本用法**：
    
    javascript
    
    运行
    
    ```
    // 创建 Promise
    const fetchData = () => {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          const success = true;
          if (success) {
            resolve({ data: '用户数据' }); // 成功，调用 resolve
          } else {
            reject(new Error('请求失败')); // 失败，调用 reject
          }
        }, 1000);
      });
    };
    
    // 使用 Promise
    fetchData()
      .then(res => console.log(res.data)) // 成功回调
      .catch(err => console.log(err.message)) // 失败回调
      .finally(() => console.log('请求完成')); // 无论成功失败都执行
    ```
    

### 2. `async/await`（ES2017）

`Promise` 的语法糖，让异步代码看起来像同步代码，更易读。

javascript

运行

```
// async 函数返回 Promise
const getData = async () => {
  try {
    // await 等待 Promise 完成
    const res = await fetchData();
    console.log(res.data); // 用户数据
  } catch (err) {
    console.log(err.message); // 捕获错误
  } finally {
    console.log('请求完成');
  }
};

getData();
```

## 七、模块化：`import` / `export`

ES6 原生支持模块化，替代了之前的 CommonJS（`require`/`module.exports`）。

### 1. 命名导出（Named Export）

可以导出多个变量 / 函数，导入时需用 `{}` 包裹。

javascript

运行

```
// module.js（导出）
export const name = '张三';
export const age = 25;
export function sayHi() {
  console.log('Hi');
}

// main.js（导入）
import { name, age, sayHi } from './module.js';
console.log(name); // 张三
sayHi(); // Hi

// 重命名导入
import { name as username } from './module.js';

// 整体导入
import * as module from './module.js';
console.log(module.name); // 张三
```

### 2. 默认导出（Default Export）

一个模块只能有一个默认导出，导入时无需 `{}`。

javascript

运行

```
// module.js（导出）
export default function add(a, b) {
  return a + b;
}

// main.js（导入）
import add from './module.js';
console.log(add(1, 2)); // 3
```

## 八、其他常用特性

### 1. 模板字符串

用反引号 `` ` `` 包裹，支持多行字符串和插值 `${}`。

javascript

运行

```
const name = '张三';
const str = `你好，${name}
这是第二行`;
console.log(str);
```

### 2. `Set` / `Map`

- **`Set`**：无重复值的集合。
    
    javascript
    
    运行
    
    ```
    const set = new Set([1, 2, 2, 3]);
    console.log(set); // Set {1, 2, 3}
    set.add(4); // 添加元素
    set.has(2); // true（判断是否包含）
    set.delete(3); // 删除元素
    ```
    
- **`Map`**：键值对集合，键可以是任意类型（Object 的键只能是字符串 / Symbol）。
    
    javascript
    
    运行
    
    ```
    const map = new Map();
    map.set('name', '张三'); // 添加键值对
    map.get('name'); // '张三'（获取值）
    map.has('name'); // true
    ```
    

### 3. `Class`（类）

语法糖，让面向对象编程更清晰。

javascript

运行

```
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  sayHi() {
    console.log(`你好，我是${this.name}`);
  }
}

// 继承
class Student extends Person {
  constructor(name, age, grade) {
    super(name, age); // 调用父类 constructor
    this.grade = grade;
  }
}

const student = new Student('李四', 18, 3);
student.sayHi(); // 你好，我是李四
```

### 4. `for...of` 循环

遍历可迭代对象（数组、`Set`、`Map` 等），可 `break`/`continue`。

javascript

运行

```
const arr = [1, 2, 3];
for (const item of arr) {
  console.log(item); // 1, 2, 3
}
```

## 总结

ES6+ 让 JavaScript 从 “玩具语言” 变成了真正的企业级开发语言，核心价值在于：

- **更简洁**：箭头函数、解构赋值、模板字符串等减少代码量。
- **更强大**：`Promise`/`async/await` 解决异步问题，`Set`/`Map` 补充数据结构。
- **更规范**：`let`/`const` 替代 `var`，模块化统一代码组织方式。