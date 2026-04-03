这是一种 **「速度优先、持久化兜底」** 的双层缓存方案，核心是利用两者的优势互补：

- **Memory Map**：内存缓存，读写速度极快（纳秒级），但页面刷新 / 关闭后数据丢失，适合做「热缓存」（高频访问的临时数据）；
- **LocalStorage**：磁盘持久化缓存，刷新 / 重启浏览器后数据仍在，但容量有限（约 5MB）、读写速度远慢于内存，适合做「冷缓存」（不常变的持久化数据）。

两者结合，既能享受内存的极致速度，又能通过 LocalStorage 实现跨会话复用，是前端高频数据缓存的经典方案。

## 一、核心设计思路

### 1. 分层定位

| 层级     | 存储介质         | 核心作用          | 优势                           | 劣势                           |
| :----- | :----------- | :------------ | :--------------------------- | :--------------------------- |
| L1 热缓存 | Memory Map   | 优先响应请求，提供极致速度 | 读写极快、无容量限制（受限于设备内存）、支持任意数据类型 | 页面刷新后丢失、非持久化                 |
| L2 冷缓存 | LocalStorage | 持久化兜底，跨会话复用数据 | 持久化存储、跨标签页 / 会话可用            | 容量有限、仅支持字符串、读写较慢、同步操作可能阻塞主线程 |
### 2. 核心原则

1. **优先读内存**：请求数据时，先查 Memory Map，命中则直接返回（最快路径）；
2. **内存未命中查 LocalStorage**：Memory Map 没有时，查 LocalStorage，命中则将数据同步到 Memory Map（为下次请求提速），再返回；
3. **都未命中则请求服务器**：拿到数据后，**同时写入 Memory Map 和 LocalStorage**，既保证当前页面的访问速度，又为下次访问做持久化兜底；
4. **过期淘汰机制**：给缓存项加过期时间，避免数据过时；LocalStorage 满时，用 LRU（最近最少使用）算法淘汰旧数据。

## 二、完整执行流程

我们以「获取用户配置」为例，完整走一遍混合缓存的流程：

1. **发起请求**：调用 `cache.get('userConfig')`；
2. **查 L1 内存缓存**：
    
    - ✅ 命中：直接返回配置数据，流程结束（耗时 < 1ms）；
    - ❌ 未命中：进入下一步；
    
3. **查 L2 持久化缓存**：
    
    - ✅ 命中且未过期：将数据从 LocalStorage 取出，JSON.parse 后存入 Memory Map，返回数据（耗时约 10-50ms）；
    - ❌ 未命中 / 已过期：进入下一步；
    
4. **请求服务器**：发起 API 请求获取用户配置；
5. **回写缓存**：
    
    - 将配置数据存入 Memory Map；
    - 将配置数据 + 过期时间戳，JSON.stringify 后存入 LocalStorage；
    
6. **返回数据**：将服务器返回的配置数据返回给调用方。

## 三、关键细节实现

### 1. 缓存项结构

为了支持过期时间和 LRU 淘汰，每个缓存项需要包含以下字段：

```javascript
{
  value: any,          // 实际缓存的数据
  expireAt: number,    // 过期时间戳（毫秒）
  lastAccessAt: number // 最后访问时间戳（用于LRU淘汰）
}
```

### 2. 过期时间处理

- 写入缓存时，根据配置的 `ttl`（生存时间，毫秒）计算 `expireAt = Date.now() + ttl`；
- 读取缓存时，判断 `Date.now() > expireAt`，若成立则删除该缓存项，视为未命中；
- 支持 `ttl: 0` 或 `ttl: Infinity` 表示永不过期（需谨慎使用）。

### 3. LRU 淘汰策略（LocalStorage 容量兜底）

LocalStorage 容量有限（约 5MB），写入时若报错（`QuotaExceededError`），需触发淘汰：

1. 遍历 LocalStorage 中所有缓存项，按 `lastAccessAt` 排序；
2. 删除最久未访问的 20% 缓存项；
3. 重新尝试写入，若仍失败则继续淘汰，直到写入成功或清空所有缓存。

### 4. 数据序列化与反序列化

- LocalStorage 仅支持字符串，写入时需用 `JSON.stringify()` 序列化；
- 读取时需用 `JSON.parse()` 反序列化；
- 注意：无法序列化函数、正则、Symbol、BigInt 等特殊类型，若需缓存此类数据，需自定义序列化逻辑（或仅用 Memory Map 缓存）。

## 四、完整代码封装

我们封装一个 `HybridCache` 类，包含完整的双层缓存、过期、淘汰逻辑：

```javascript
class HybridCache {
  /**
   * 构造函数
   * @param {Object} options - 配置项
   * @param {string} options.prefix - LocalStorage 键名前缀，避免冲突
   * @param {number} options.defaultTTL - 默认缓存生存时间（毫秒），默认1小时
   */
  constructor(options = {}) {
    this.prefix = options.prefix || 'hybrid_cache_';
    this.defaultTTL = options.defaultTTL || 3600 * 1000;
    // L1 内存缓存：Map 结构，支持任意键值
    this.memoryCache = new Map();
  }

  /**
   * 生成 LocalStorage 键名
   * @param {string} key - 缓存键
   * @returns {string} 带前缀的键名
   */
  _getStorageKey(key) {
    return `${this.prefix}${key}`;
  }

  /**
   * 从 LocalStorage 读取所有缓存项（用于淘汰）
   * @returns {Array} 缓存项数组
   */
  _getAllStorageItems() {
    const items = [];
    for (let i = 0; i < localStorage.length; i++) {
      const fullKey = localStorage.key(i);
      if (fullKey.startsWith(this.prefix)) {
        try {
          const value = JSON.parse(localStorage.getItem(fullKey));
          items.push({ key: fullKey, ...value });
        } catch (e) {
          // 解析失败，删除无效项
          localStorage.removeItem(fullKey);
        }
      }
    }
    return items;
  }

  /**
   * LRU 淘汰 LocalStorage 旧数据
   */
  _evictStorage() {
    const items = this._getAllStorageItems();
    // 按最后访问时间排序，最久的在前
    items.sort((a, b) => a.lastAccessAt - b.lastAccessAt);
    // 删除最久的 20%
    const toDelete = items.slice(0, Math.ceil(items.length * 0.2));
    toDelete.forEach(item => {
      localStorage.removeItem(item.key);
      // 同步删除内存缓存
      const originalKey = item.key.replace(this.prefix, '');
      this.memoryCache.delete(originalKey);
    });
  }

  /**
   * 写入 LocalStorage（带容量兜底）
   * @param {string} key - 缓存键
   * @param {Object} value - 缓存项
   */
  _setStorage(key, value) {
    const fullKey = this._getStorageKey(key);
    const serialized = JSON.stringify(value);
    let retry = 3;
    while (retry > 0) {
      try {
        localStorage.setItem(fullKey, serialized);
        return;
      } catch (e) {
        if (e.name === 'QuotaExceededError') {
          this._evictStorage();
          retry--;
        } else {
          throw e;
        }
      }
    }
    console.warn('HybridCache: LocalStorage 写入失败，容量已满');
  }

  /**
   * 获取缓存
   * @param {string} key - 缓存键
   * @returns {any|null} 缓存数据，未命中/过期返回 null
   */
  get(key) {
    const now = Date.now();

    // 1. 优先查 L1 内存缓存
    if (this.memoryCache.has(key)) {
      const item = this.memoryCache.get(key);
      // 判断是否过期
      if (item.expireAt > now) {
        // 更新最后访问时间
        item.lastAccessAt = now;
        this.memoryCache.set(key, item);
        return item.value;
      } else {
        // 过期，删除内存缓存
        this.memoryCache.delete(key);
      }
    }

    // 2. 查 L2 LocalStorage
    const fullKey = this._getStorageKey(key);
    const storageValue = localStorage.getItem(fullKey);
    if (storageValue) {
      try {
        const item = JSON.parse(storageValue);
        if (item.expireAt > now) {
          // 未过期：同步到内存缓存，更新最后访问时间
          item.lastAccessAt = now;
          this.memoryCache.set(key, item);
          this._setStorage(key, item);
          return item.value;
        } else {
          // 过期：删除 LocalStorage 缓存
          localStorage.removeItem(fullKey);
        }
      } catch (e) {
        // 解析失败，删除无效项
        localStorage.removeItem(fullKey);
      }
    }

    // 3. 都未命中
    return null;
  }

  /**
   * 设置缓存
   * @param {string} key - 缓存键
   * @param {any} value - 缓存数据
   * @param {number} [ttl] - 生存时间（毫秒），不传则用默认值
   */
  set(key, value, ttl) {
    const now = Date.now();
    const expireAt = now + (ttl || this.defaultTTL);
    const item = {
      value,
      expireAt,
      lastAccessAt: now
    };

    // 1. 写入 L1 内存缓存
    this.memoryCache.set(key, item);
    // 2. 写入 L2 LocalStorage
    this._setStorage(key, item);
  }

  /**
   * 删除指定缓存
   * @param {string} key - 缓存键
   */
  delete(key) {
    this.memoryCache.delete(key);
    localStorage.removeItem(this._getStorageKey(key));
  }

  /**
   * 清空所有缓存
   */
  clear() {
    this.memoryCache.clear();
    // 只删除带前缀的项，避免误删其他数据
    const keysToDelete = [];
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i);
      if (key.startsWith(this.prefix)) {
        keysToDelete.push(key);
      }
    }
    keysToDelete.forEach(key => localStorage.removeItem(key));
  }
}

// 导出实例（单例模式）
export default new HybridCache({
  prefix: 'my_app_cache_',
  defaultTTL: 3600 * 1000 // 默认1小时
});
```

## 五、使用示例

### 1. 基础使用

```javascript
import cache from './hybrid-cache.js';

// 设置缓存（默认1小时过期）
cache.set('userInfo', { id: 1, name: '张三' });

// 设置缓存（自定义过期时间：10分钟）
cache.set('tempData', { a: 1 }, 10 * 60 * 1000);

// 获取缓存
const userInfo = cache.get('userInfo');
if (userInfo) {
  console.log('命中缓存：', userInfo);
} else {
  console.log('未命中缓存，请求服务器...');
  // 发起 API 请求
  // const data = await fetchUserInfo();
  // cache.set('userInfo', data);
}

// 删除缓存
cache.delete('tempData');

// 清空所有缓存
cache.clear();
```
### 2. 结合 API 请求的完整示例

```javascript
import cache from './hybrid-cache.js';

// 获取用户配置（优先读缓存）
async function getUserConfig() {
  const cacheKey = 'user_config';
  // 1. 查缓存
  const cached = cache.get(cacheKey);
  if (cached) {
    return cached;
  }
  // 2. 未命中，请求服务器
  const res = await fetch('/api/user/config');
  const data = await res.json();
  // 3. 回写缓存（缓存24小时）
  cache.set(cacheKey, data, 24 * 3600 * 1000);
  return data;
}
```

## 六、适用场景与注意事项

### 1. 适用场景

- **用户信息、配置项**：不常变、需要跨会话复用的数据；
- **静态列表数据**：比如商品分类、城市列表、字典数据；
- **高频访问的小体积数据**：比如搜索历史、最近浏览记录；
- **离线可用数据**：配合 Service Worker，实现弱网 / 离线场景下的基础功能。

### 2. 注意事项

- **LocalStorage 容量限制**：仅适合缓存小体积数据（<1MB），大体积数据建议用 IndexedDB；
- **特殊数据类型**：LocalStorage 无法序列化函数、正则、Symbol 等，此类数据仅用 Memory Map 缓存；
- **数据一致性**：若服务器数据更新，需主动删除对应缓存（或设置合理的 TTL），避免用户看到旧数据；
- **跨标签页同步**：LocalStorage 支持跨标签页共享，但 Memory Map 不支持，可监听 `storage` 事件，同步更新内存缓存：
    
```javascript
    window.addEventListener('storage', (e) => {
      if (e.key.startsWith(cache.prefix)) {
        const key = e.key.replace(cache.prefix, '');
        if (e.newValue) {
          // 有新值，同步到内存
          cache.memoryCache.set(key, JSON.parse(e.newValue));
        } else {
          // 被删除，同步删除内存
          cache.memoryCache.delete(key);
        }
      }
    });
```