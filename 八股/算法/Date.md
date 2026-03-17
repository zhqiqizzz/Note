`Date` 用于处理日期和时间，需用 `new Date()` 创建实例。
### 1. 创建日期实例

```javascript
// 当前时间
const now = new Date();
// 指定时间（年, 月(0-11), 日, 时, 分, 秒）
const specific = new Date(2024, 0, 1, 12, 0, 0); // 2024年1月1日12:00:00
// 时间戳
const timestamp = new Date(1704067200000);
// 日期字符串
const dateStr = new Date('2024-01-01');
```

### 2. 获取日期 / 时间

|方法|作用|示例（2024-01-01 12:30:45）|
|---|---|---|
|`getFullYear()`|获取年份（4 位）|`2024`|
|`getMonth()`|获取月份（0-11，0 是 1 月）|`0`|
|`getDate()`|获取日期（1-31）|`1`|
|`getDay()`|获取星期（0-6，0 是周日）|`1`|
|`getHours()`|获取小时（0-23）|`12`|
|`getMinutes()`|获取分钟（0-59）|`30`|
|`getSeconds()`|获取秒（0-59）|`45`|
|`getTime()`|获取时间戳（毫秒）|`1704067845000`|
### 3. 设置日期 / 时间

对应 `getXxx()` 有 `setXxx()` 方法，如 `setFullYear()`、`setMonth()` 等。

### 4. 格式化日期

```javascript
// 简单格式化：2024-01-01 12:30:45
function formatDate(date) {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  const hours = String(date.getHours()).padStart(2, '0');
  const minutes = String(date.getMinutes()).padStart(2, '0');
  const seconds = String(date.getSeconds()).padStart(2, '0');
  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
}
console.log(formatDate(new Date()));
```