
`call` 和 `apply` 是 JavaScript 中 `Function.prototype` 上的原生方法，它们的核心作用完全一致：**在调用函数时，显式指定函数内部 `this` 的指向**，同时执行该函数。二者的唯一区别仅在于参数传递的形式不同。

## call 方法

`call` 方法的语法为：`func.call(thisArg, arg1, arg2, ...)`。

- 第一个参数 `thisArg`：函数执行时 `this` 要指向的对象。若传入 `null` 或 `undefined`，在非严格模式下 `this` 会指向全局对象（浏览器中为 `window`），严格模式下则为 `undefined`。
- 后续参数：函数执行所需的参数，需**逐个传入**。

### 示例：改变 this 指向

```javascript
const person = {
  name: 'Alice',
  greet: function(greeting, punctuation) {
    console.log(`${greeting}, I'm ${this.name}${punctuation}`);
  }
};

const anotherPerson = { name: 'Bob' };
// 调用 greet 方法，将 this 指向 anotherPerson，并逐个传参
person.greet.call(anotherPerson, 'Hello', '!'); 
// 输出：Hello, I'm Bob!
```

### 示例：借用原生方法

类数组对象（如 `arguments`、DOM 节点列表）没有数组的 `slice` 等方法，可通过 `call` 借用数组方法：

```javascript
function getArgs() {
  // 借用 Array.prototype.slice 将 arguments 转为数组
  return Array.prototype.slice.call(arguments);
}
console.log(getArgs(1, 2, 3)); // 输出：[1, 2, 3]
```

## apply 方法

`apply` 方法的语法为：`func.apply(thisArg, [argsArray])`。

- 第一个参数 `thisArg`：与 `call` 一致，指定 `this` 的指向。
- 第二个参数：函数执行所需的参数，需以**数组或类数组对象**的形式传入。

### 示例：与 call 对比传参

同样是上面的 `greet` 方法，用 `apply` 调用时参数需放在数组中：

```javascript
const person = {
  name: 'Alice',
  greet: function(greeting, punctuation) {
    console.log(`${greeting}, I'm ${this.name}${punctuation}`);
  }
};

const anotherPerson = { name: 'Bob' };
// 调用 greet 方法，this 指向 anotherPerson，参数以数组形式传入
person.greet.apply(anotherPerson, ['Hi', '~']); 
// 输出：Hi, I'm Bob~
```

### 示例：数学运算中的应用

利用 `apply` 可将数组直接作为参数传递给不接受数组的方法，比如求数组最大值：

```javascript
const numbers = [3, 7, 2, 9, 5];
// Math.max 原本只接受逐个参数，用 apply 可传入数组
const max = Math.max.apply(null, numbers);
console.log(max); // 输出：9
```

## call 与 apply 的区别

二者的核心区别仅在于**参数传递方式**：

- `call`：参数逐个传入，适合已知参数数量的场景。
- `apply`：参数以数组或类数组形式传入，适合参数是数组、或参数数量不确定的场景。

在性能上，现代 JavaScript 引擎中二者差异极小，通常可忽略，主要根据参数形式选择即可。

## 常见应用场景

1. **实现构造函数继承**：在子构造函数中通过 `call` 调用父构造函数，继承父构造函数的属性：

```javascript
function Parent(name) {
  this.name = name;
}
function Child(name, age) {
  // 调用 Parent 构造函数，将 this 指向 Child 实例
  Parent.call(this, name);
  this.age = age;
}
const child = new Child('Tom', 5);
console.log(child.name); // 输出：Tom
```

2. **借用对象方法**：如前面示例中，让一个对象借用另一个对象的方法，无需重复定义。
3. **处理类数组对象**：将 `arguments`、DOM 节点列表等类数组转为数组，或借用数组的遍历、拼接等方法。