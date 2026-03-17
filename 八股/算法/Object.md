JavaScript 中的 `Object` 是所有对象的基础，提供了大量实用的**静态方法**（直接通过 `Object.xxx` 调用）和**实例方法**（通过对象实例调用）。以下是最常用的方法分类整理：

### 一、对象创建与原型操作

#### 1. `Object.create(proto, [propertiesObject])`

- **作用**：基于指定原型创建一个新对象，可同时定义新对象的属性。
- **示例**：

```javascript
    const person = { name: 'Alice' };
    const student = Object.create(person, {
      age: { value: 18, writable: true, enumerable: true }
    });
    console.log(student.name); // 'Alice'（继承自 person）
    console.log(student.age);  // 18
```

#### 2. `Object.getPrototypeOf(obj)`

- **作用**：获取对象的原型（即内部 `[[Prototype]]` 属性）。
- **示例**：

```javascript
    const arr = [];
    console.log(Object.getPrototypeOf(arr) === Array.prototype); // true
```

#### 3. `Object.setPrototypeOf(obj, proto)`

- **作用**：设置对象的原型（性能较差，建议优先用 `Object.create()`）。
- **示例**：

```javascript
    const obj = {};
    Object.setPrototypeOf(obj, Array.prototype);
    console.log(obj.push); // [Function: push]（继承自 Array）
```

### 二、属性操作

#### 1. `Object.defineProperty(obj, prop, descriptor)`

- **作用**：定义或修改对象的单个属性特性（如是否可写、可枚举等）。
- **属性描述符**：
    
    - `value`：属性值（默认 `undefined`）。
    - `writable`：是否可修改（默认 `false`）。
    - `enumerable`：是否可枚举（默认 `false`）。
    - `configurable`：是否可删除 / 修改描述符（默认 `false`）。
    
- **示例**：
    
```javascript
    const obj = {};
    Object.defineProperty(obj, 'name', {
      value: 'Bob',
      writable: false,    // 不可修改
      enumerable: true,   // 可枚举
      configurable: true  // 可删除
    });
    obj.name = 'Alice';   // 无效（writable: false）
    console.log(obj.name); // 'Bob'
```

#### 2. `Object.defineProperties(obj, props)`

- **作用**：批量定义或修改多个属性的特性。
- **示例**：
    
```javascript
    const obj = {};
    Object.defineProperties(obj, {
      name: { value: 'Alice', enumerable: true },
      age: { value: 25, writable: true }
    });
```

#### 3. `Object.getOwnPropertyDescriptor(obj, prop)`

- **作用**：获取对象某个自身属性的描述符。
- **示例**：

```javascript
    const obj = { name: 'Alice' };
    console.log(Object.getOwnPropertyDescriptor(obj, 'name'));
    // 输出：{ value: 'Alice', writable: true, enumerable: true, configurable: true }
```

#### 4. `Object.hasOwn(obj, prop)`（推荐）/ `obj.hasOwnProperty(prop)`

- **作用**：判断属性是否为对象**自身属性**（非继承）。
- **区别**：`Object.hasOwn()` 是 ES2022 新方法，更安全（避免对象重写 `hasOwnProperty`）。
- **示例**：

```javascript
    const obj = { name: 'Alice' };
    console.log(Object.hasOwn(obj, 'name'));        // true
    console.log(Object.hasOwn(obj, 'toString'));    // false（继承自原型）
```

### 三、遍历与键值获取

#### 1. `Object.keys(obj)`

- **作用**：返回对象**自身可枚举属性**的键组成的数组。
- **示例**：

```javascript
    const obj = { name: 'Alice', age: 25 };
    console.log(Object.keys(obj)); // ['name', 'age']
```

#### 2. `Object.values(obj)`

- **作用**：返回对象**自身可枚举属性**的值组成的数组。
- **示例**：
    
```javascript
    const obj = { name: 'Alice', age: 25 };
    console.log(Object.values(obj)); // ['Alice', 25]
```

#### 3. `Object.entries(obj)`

- **作用**：返回对象**自身可枚举属性**的 `[key, value]` 对数组。
- **示例**：
    
```javascript
    const obj = { name: 'Alice', age: 25 };
    console.log(Object.entries(obj)); // [['name', 'Alice'], ['age', 25]]
    
    // 配合 for...of 遍历
    for (const [key, value] of Object.entries(obj)) {
      console.log(`${key}: ${value}`);
    }
```

#### 4. `Object.getOwnPropertyNames(obj)`

- **作用**：返回对象**所有自身属性**的键数组（包括不可枚举属性，不含 `Symbol` 键）。
- **示例**：
    
```javascript
    const obj = { name: 'Alice' };
    Object.defineProperty(obj, 'age', { value: 25, enumerable: false });
    console.log(Object.getOwnPropertyNames(obj)); // ['name', 'age']（包含不可枚举的 'age'）
```

### 四、对象合并与拷贝

#### 1. `Object.assign(target, ...sources)`

- **作用**：将一个或多个源对象的**自身可枚举属性**浅拷贝到目标对象，返回目标对象。
- **注意**：浅拷贝（仅复制对象引用，深层对象仍共享）。
- **示例**：
    
```javascript
    const target = { a: 1 };
    const source1 = { b: 2 };
    const source2 = { c: 3, a: 4 }; // 同名属性会覆盖
    const result = Object.assign(target, source1, source2);
    console.log(result); // { a: 4, b: 2, c: 3 }
    console.log(target === result); // true（直接修改 target）
```

### 五、对象状态控制（冻结、密封）

#### 1. `Object.freeze(obj)`

- **作用**：**冻结**对象，使其不可添加、删除、修改属性（浅冻结，深层对象仍可修改）。
- **示例**：
    
```javascript
    const obj = { name: 'Alice', info: { age: 25 } };
    Object.freeze(obj);
    obj.name = 'Bob';       // 无效（冻结后不可修改）
    obj.info.age = 30;      // 有效（深层对象未冻结）
    console.log(obj.name);   // 'Alice'
    console.log(obj.info.age); // 30
```

#### 2. `Object.isFrozen(obj)`

- **作用**：判断对象是否被冻结。
- **示例**：
    
```javascript
    const obj = {};
    Object.freeze(obj);
    console.log(Object.isFrozen(obj)); // true
```

#### 3. `Object.seal(obj)`

- **作用**：**密封**对象，使其不可添加、删除属性，但可修改现有属性值。
- **示例**：

```javascript
    const obj = { name: 'Alice' };
    Object.seal(obj);
    obj.name = 'Bob';       // 有效（可修改）
    delete obj.name;         // 无效（不可删除）
    obj.age = 25;            // 无效（不可添加）
```

#### 4. `Object.isSealed(obj)`

- **作用**：判断对象是否被密封。

### 六、其他实用方法

#### 1. `Object.is(value1, value2)`

- **作用**：判断两个值是否**严格相等**，解决了 `===` 的两个特殊情况：
    
    - `NaN === NaN` 为 `false`，但 `Object.is(NaN, NaN)` 为 `true`。
    - `+0 === -0` 为 `true`，但 `Object.is(+0, -0)` 为 `false`。
    
- **示例**：

```javascript
    console.log(Object.is(NaN, NaN));   // true
    console.log(Object.is(+0, -0));     // false
    console.log(Object.is('a', 'a'));   // true
```

### 七、实例方法（原型上的方法）

#### 1. `obj.toString()`

- **作用**：返回对象的字符串表示。常通过 `Object.prototype.toString.call()` 判断数据类型。
- **示例**：

```javascript
    const arr = [1, 2, 3];
    console.log(arr.toString()); // '1,2,3'（Array 重写了 toString）
    console.log(Object.prototype.toString.call(arr)); // '[object Array]'（判断类型）
    console.log(Object.prototype.toString.call(null)); // '[object Null]'
```
#### 2. `obj.valueOf()`

- **作用**：返回对象的原始值。通常自动调用（如对象参与运算时）。
- **示例**：
    
```javascript
    const obj = { value: 10 };
    obj.valueOf = function() { return this.value; };
    console.log(obj + 5); // 15（自动调用 valueOf）
```