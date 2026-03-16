`this` 是 JavaScript 函数执行时的**上下文对象**，它的指向完全取决于函数的**调用方式**，而非定义位置。理解 `this` 的关键是：**谁调用了函数，this 就指向谁**（箭头函数除外）。

## 不同场景下的 this 指向

### 1. 全局环境中调用函数

在全局作用域直接调用普通函数时，`this` 的指向分两种情况：

- **非严格模式**：`this` 指向全局对象（浏览器中为 `window`，Node.js 中为 `global`）。
- **严格模式**（`'use strict'`）：`this` 为 `undefined`。

```javascript
// 非严格模式
function globalFunc() {
  console.log(this);
}
globalFunc(); // 输出：window（浏览器环境）

// 严格模式
'use strict';
function strictFunc() {
  console.log(this);
}
strictFunc(); // 输出：undefined
```

### 2. 作为对象方法调用

当函数作为某个对象的方法被调用时，`this` 指向**调用该方法的对象**。

```javascript
const person = {
  name: 'Alice',
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
};
person.greet(); // 输出：Hello, I'm Alice（this 指向 person）
```

⚠️ 易踩坑：如果将对象方法赋值给变量后单独调用，`this` 会变回全局对象（非严格模式）：

```javascript
const fn = person.greet;
fn(); // 输出：Hello, I'm undefined（this 指向 window，window.name 为空）
```

### 3. 作为构造函数调用（使用 new）

用 `new` 关键字调用构造函数时，`this` 指向**新创建的实例对象**。

```javascript
function Person(name) {
  this.name = name; // this 指向新实例
}
const alice = new Person('Alice');
console.log(alice.name); // 输出：Alice
```

### 4. 通过 call/apply/bind 显式指定 this

`call`、`apply`、`bind` 是 `Function.prototype` 上的方法，可**强制指定函数内部 `this` 的指向**：

- `call(thisArg, arg1, arg2, ...)`：立即执行函数，`this` 指向 `thisArg`，参数逐个传入。
- `apply(thisArg, [argsArray])`：立即执行函数，`this` 指向 `thisArg`，参数以数组形式传入。
- `bind(thisArg)`：返回一个新函数，`this` 永久绑定为 `thisArg`，不会再被其他方式改变。

```javascript
const person1 = { name: 'Alice' };
const person2 = { name: 'Bob' };

function greet() {
  console.log(`Hello, I'm ${this.name}`);
}

greet.call(person1); // 输出：Hello, I'm Alice
greet.apply(person2); // 输出：Hello, I'm Bob

const boundGreet = greet.bind(person1);
boundGreet(); // 输出：Hello, I'm Alice（永久绑定 person1）
```

### 5. 箭头函数中的 this

箭头函数是 ES6 新增的语法，它**不绑定自己的 `this`**，而是**继承外层作用域（词法作用域）的 `this`**，且指向一旦确定就不会改变。

```javascript
const person = {
  name: 'Alice',
  greetLater() {
    // 普通函数作为 setTimeout 回调，this 指向 window（非严格模式）
    setTimeout(function() {
      console.log(`Hello, I'm ${this.name}`);
    }, 1000);
    
    // 箭头函数继承外层 greetLater 的 this（即 person）
    setTimeout(() => {
      console.log(`Hello, I'm ${this.name}`);
    }, 1000);
  }
};
person.greetLater();
// 1秒后输出：
// Hello, I'm undefined（普通函数）
// Hello, I'm Alice（箭头函数）
```

## 总结：this 指向判断规则

1. 箭头函数：继承外层作用域的 `this`。
2. `new` 调用：`this` 指向新创建的实例。
3. `call`/`apply`/`bind` 调用：`this` 指向指定的对象。
4. 对象方法调用：`this` 指向调用方法的对象。
5. 普通函数调用：非严格模式下指向全局对象，严格模式下为 `undefined`。