Map 是 ES6 新增的**键值对集合**，键可以是任意类型（对象、函数、基本类型等），以下是核心常用方法：

#### 1. 增删改查

- **`set(key, value)`**：添加或更新键值对，返回 Map 本身（支持链式调用）。

```javascript
    const map = new Map();
    map.set('name', 'Alice').set(1, 'one');
```

- **`get(key)`**：获取键对应的值，键不存在返回 `undefined`。
- **`has(key)`**：判断是否包含某个键，返回布尔值。
- **`delete(key)`**：删除指定键值对，返回布尔值表示是否删除成功。
- **`clear()`**：清空所有键值对，无返回值。

#### 2. 遍历与迭代

- **`keys()`**：返回所有键的迭代器。
- **`values()`**：返回所有值的迭代器。
- **`entries()`**：返回所有 `[key, value]` 键值对的迭代器（Map 默认迭代方式）。
- **`forEach(callback)`**：遍历每个键值对，回调参数为 `(value, key, map)`。

#### 3. 其他

- **`size`**：属性，返回 Map 中键值对的数量（不是方法，直接访问）。

### 简单示例

```javascript
const map = new Map([['a', 1], ['b', 2]]);

map.set('c', 3);     // Map(3) {'a'=>1, 'b'=>2, 'c'=>3}
map.get('a');         // 1
map.has('d');         // false
map.size;             // 3

// 遍历
for (const [k, v] of map) {
  console.log(k, v);  // 输出 a 1, b 2, c 3
}
```