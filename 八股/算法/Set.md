JavaScript 的 `Set` 是一种**无重复值**的有序集合，它提供了一系列实用的方法来操作和遍历元素。下面我把最常用的方法和属性结合例子给你梳理一遍：

### 一、核心属性

- **`size`**：返回 Set 中元素的个数（类似数组的 `length`）。
    
```javascript
    const set = new Set([1, 2, 3]);
    console.log(set.size); // 输出 3
```

### 二、核心操作方法

1. **`add(value)`**：向 Set 中添加一个元素，返回 Set 本身（支持链式调用）。
    
```javascript
    const set = new Set();
    set.add(1).add(2).add(2); // 重复的 2 会被忽略
    console.log(set); // Set { 1, 2 }
```
    
2. **`has(value)`**：判断某个值是否在 Set 中，返回布尔值 `true/false`。
    
```javascript
    const set = new Set([1, 2, 3]);
    console.log(set.has(2)); // true
    console.log(set.has(4)); // false
```
    
3. **`delete(value)`**：删除 Set 中的某个值，返回布尔值表示是否删除成功。
    
```javascript
    const set = new Set([1, 2, 3]);
    console.log(set.delete(2)); // true（删除成功）
    console.log(set.delete(4)); // false（值不存在）
    console.log(set); // Set { 1, 3 }
```
    
4. **`clear()`**：清空 Set 中的所有元素，无返回值。
    
```javascript
    const set = new Set([1, 2, 3]);
    set.clear();
    console.log(set.size); // 0
```

### 三、遍历方法

Set 是可迭代对象，可以用 `for...of` 直接遍历，也提供了以下方法获取迭代器：

1. **`values()`**：返回一个包含所有值的迭代器（最常用）。
    
```javascript
    const set = new Set(['a', 'b', 'c']);
    for (const val of set.values()) {
        console.log(val); // 依次输出 'a', 'b', 'c'
    }
```
    
2. **`keys()`**：和 `values()` 完全一样（因为 Set 只有值，没有键）。
    
```javascript
    const set = new Set(['a', 'b']);
    console.log([...set.keys()]); // ['a', 'b']
```
    
3. **`entries()`**：返回包含 `[value, value]` 键值对的迭代器（为了和 Map 保持一致）。
    
```javascript
    const set = new Set(['a', 'b']);
    for (const [key, val] of set.entries()) {
        console.log(key, val); // 输出 'a a', 'b b'
    }
```
    
4. **`forEach(callback)`**：遍历 Set 中的每个元素，执行回调函数。
    
```javascript
    const set = new Set([1, 2, 3]);
    set.forEach((val, key, set) => {
        console.log(val); // 依次输出 1, 2, 3
    });
```

### 使用场景
#### 场景 1：数组去重（最经典、最高频）

这是 Set 最广为人知的用法，利用 Set **自动忽略重复值**的特性，一行代码完成数组去重，比遍历判断的写法简洁高效得多。

```javascript
// 基础数组去重
const arr = [1, 2, 2, 3, 3, 3, 4];
const uniqueArr = [...new Set(arr)]; // 用展开运算符转回数组
console.log(uniqueArr); // 输出 [1, 2, 3, 4]

// 也可以用 Array.from 转换
const uniqueArr2 = Array.from(new Set(arr));
console.log(uniqueArr2); // 同样输出 [1, 2, 3, 4]
```

Set 是通过 **严格相等（`===`）** 判断值是否重复的，所以对**引用类型（对象、数组）** 无法直接去重（因为引用地址不同，即使内容一样也会被判定为不同值）。

#### 场景 2：数组的集合运算（交集 / 并集 / 差集）

处理两个数组的公共元素、合并去重、独有元素时，用 Set 实现比数组遍历更简洁，性能也更好。
##### 1. 并集：合并两个数组，保留所有不重复的元素

```javascript
const arr1 = [1, 2, 3];
const arr2 = [3, 4, 5];
const union = [...new Set([...arr1, ...arr2])];
console.log(union); // 输出 [1, 2, 3, 4, 5]
```
##### 2. 交集：两个数组都包含的公共元素

```javascript
const arr1 = [1, 2, 3];
const arr2 = [3, 4, 5];
const set1 = new Set(arr1);
const set2 = new Set(arr2);
// 保留 set1 里同时存在于 set2 的元素
const intersection = [...set1].filter(item => set2.has(item));
console.log(intersection); // 输出 [3]
```
##### 3. 差集：存在于第一个数组、但不存在于第二个数组的元素

```javascript
const arr1 = [1, 2, 3];
const arr2 = [3, 4, 5];
const set1 = new Set(arr1);
const set2 = new Set(arr2);
// 保留 set1 里不在 set2 中的元素
const difference = [...set1].filter(item => !set2.has(item));
console.log(difference); // 输出 [1, 2]
```