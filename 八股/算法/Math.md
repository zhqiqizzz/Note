`Math` 是一个内置对象，提供数学常量和方法，无需 `new`，直接调用。

### 1. 常用方法

| 方法                 | 作用               | 示例                      |
| ------------------ | ---------------- | ----------------------- |
| `Math.random()`    | 返回 [0, 1) 之间的随机数 | `Math.random()` → 0.123 |
| `Math.floor(x)`    | 向下取整             | `Math.floor(3.9)` → 3   |
| `Math.ceil(x)`     | 向上取整             | `Math.ceil(3.1)` → 4    |
| `Math.round(x)`    | 四舍五入             | `Math.round(3.5)` → 4   |
| `Math.max(...arr)` | 求最大值             | `Math.max(1, 3, 2)` → 3 |
| `Math.min(...arr)` | 求最小值             | `Math.min(1, 3, 2)` → 1 |
| `Math.abs(x)`      | 绝对值              | `Math.abs(-5)` → 5      |
| `Math.pow(x, y)`   | x 的 y 次方         | `Math.pow(2, 3)` → 8    |
| `Math.sqrt(x)`     | 平方根              | `Math.sqrt(9)` → 3      |
### 2. 实战：生成指定范围的随机数

```javascript
// 生成 [min, max] 之间的随机整数
function randomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
console.log(randomInt(1, 10)); // 1-10之间的随机整数
```