行业内俗称的「SKU - 幂集」，是电商场景中**处理多规格 SKU 的核心算法**，它包含两个核心应用：

1. 基于**笛卡尔积**（常被俗称 “幂集生成”）实现 SKU 全组合自动生成（用于商品发布后台）；
2. 基于**数学幂集**实现 SKU 规格选择器的可选 / 禁用状态实时判断（用于商品详情页，也是「SKU 幂集算法」的核心由来）。

下面结合电商业务场景，从基础概念到代码实现完整讲解。

## 一、基础概念铺垫

### 1. 什么是 SKU（电商场景）

SKU = Stock Keeping Unit（库存保有单位），是电商中区分商品不同售卖规格的**最小单元**，对应唯一的库存、价格、货号。

> 示例：一件 T 恤有「颜色：红 / 白」、「尺码：S/M」2 组规格，那么「红色 + S 码」「白色 + M 码」就是 2 个独立的 SKU，各自对应独立的库存和价格。

### 2. 什么是幂集（数学定义）

幂集是指**一个集合的所有子集组成的集合**，包含空集和原集合本身。

- 数学规律：若原集合有 `n` 个元素，它的幂集元素个数为 `2ⁿ`。
- 示例：
    
    原集合 `A = ['红色', 'S码']`
    
    它的幂集 `P(A) = [ [], ['红色'], ['S码'], ['红色', 'S码'] ]`
    
    共 `2²=4` 个子集，完全符合规律。

### 3. 幂集在 SKU 场景的核心价值

电商 SKU 的两大核心痛点，正好由幂集 / 笛卡尔积解决：

|业务场景|核心算法|解决的问题|
|---|---|---|
|商品发布后台|多集合笛卡尔积（俗称 “幂集生成”）|运营选择多组规格后，自动生成所有可能的 SKU 组合，无需手动新增|
|商品详情页规格选择|数学幂集|用户选规格时，实时判断哪些选项有库存、哪些需要禁用，避免用户选到无库存组合|
## 二、场景 1：SKU 全组合生成（笛卡尔积，俗称 SKU 生成）

这是电商后台商品发布页的核心功能，运营只需选择规格组（如颜色、尺码、材质）和对应的选项，系统自动生成所有规格的全组合 SKU。

### 1. 核心原理

本质是**多集合的笛卡尔积**：从每个规格组中取出一个选项，组合成一个唯一的 SKU，所有可能的组合就是笛卡尔积的结果。

> 示例：
> 
> 规格组 1（颜色）：[' 红 ', ' 白 ']
> 
> 规格组 2（尺码）：['S', 'M']
> 
> 笛卡尔积结果（全 SKU 组合）：
> 
> `[ ['红','S'], ['红','M'], ['白','S'], ['白','M'] ]`
> 
> 共 `2*2=4` 个 SKU，和业务预期完全一致。

### 2. 代码实现（JavaScript 通用）

```javascript
/**
 * 生成多规格SKU全组合（笛卡尔积）
 * @param {Array} specList 规格组列表，格式：[{ name: '颜色', options: ['红', '白'] }, ...]
 * @returns {Array} 完整SKU列表，带唯一ID、规格、价格、库存字段
 */
function generateSkuCombination(specList) {
  // 提取所有规格的选项数组
  const optionsList = specList.map(spec => spec.options);
  
  // 递归计算笛卡尔积核心逻辑
  const calcCartesian = (arrays) => {
    if (arrays.length === 0) return [[]];
    const [first, ...rest] = arrays;
    const restResult = calcCartesian(rest);
    // 拼接当前项和剩余项的所有组合
    return first.flatMap(item => 
      restResult.map(restItem => [item, ...restItem])
    );
  };

  // 生成原始组合
  const combinations = calcCartesian(optionsList);

  // 格式化业务可用的SKU结构
  return combinations.map(combination => {
    const skuSpec = {};
    specList.forEach((spec, index) => {
      skuSpec[spec.name] = combination[index];
    });
    return {
      skuId: Math.random().toString(36).substring(2, 10), // 生成唯一SKU ID
      spec: skuSpec,
      price: 0, // 运营后续填写价格
      stock: 0, // 运营后续填写库存
      disabled: false // 标记是否启用
    };
  });
}

// 测试示例
const specList = [
  { name: '颜色', options: ['红', '白'] },
  { name: '尺码', options: ['S', 'M', 'L'] },
  { name: '材质', options: ['棉', '麻'] }
];
const skuList = generateSkuCombination(specList);
console.log(skuList); // 输出 2*3*2=12 个完整SKU
```

### 3. 注意事项

- 规格数量限制：笛卡尔积结果是指数级增长的，比如 5 组规格、每组 5 个选项，会生成 `5^5=3125` 个 SKU，远超运营可维护范围，主流电商一般限制规格组≤3 组，每组选项≤10 个。
- 空规格过滤：如果某组规格的选项为空，需提前过滤，否则会生成无效空组合。

## 三、场景 2：SKU 规格选择器的幂集算法（核心应用）

这是商品详情页的核心功能，也是「SKU 幂集算法」的真正由来。用户选择规格时，需要实时判断未选规格的选项是否有库存、是否需要禁用，幂集算法是目前该场景的最优解。

### 1. 传统方案的痛点

传统方案是每次用户选规格后，遍历所有 SKU 匹配已选规格，再筛选出可选选项。但当 SKU 数量多、规格组多时，性能差、逻辑复杂，极易出现边界 bug。

### 2. 幂集算法的核心原理

**核心思路**：提前把所有有库存的 SKU，生成其规格的所有子集（幂集），存入一个「路径字典 Map」；后续用户选规格时，直接查字典即可判断是否可选，时间复杂度从 O (n) 降到 O (1)。

#### 具体执行步骤

1. **预处理阶段（仅执行 1 次）**：
    
    - 过滤掉无库存的 SKU；
    - 把每个有库存 SKU 的规格，转为排序后的键值对数组（保证序列化顺序一致）；
    - 生成该数组的幂集（所有子集），每个子集序列化为字符串，作为 Map 的 key，value 标记为 true（代表该路径有库存）。
    
2. **实时判断阶段（用户每次选规格时执行）**：
    
    - 对于未选规格的每个选项，把「用户已选规格」和「当前选项」合并成临时组合；
    - 把临时组合序列化为字符串，去预处理好的 Map 中查询；
    - 存在则该选项可选，不存在则禁用。
    

### 3. 关键细节：规格序列化

必须保证规格的顺序一致，比如 `['颜色:红', '尺码:S']` 和 `['尺码:S', '颜色:红']` 要序列化为同一个 key，否则会判断错误。

- 规范做法：先按规格名称字典序排序，再用特殊分隔符（如`|`，避免和规格内容冲突）拼接。

### 4. 完整代码实现

```javascript
/**
 * 生成数组的幂集（所有子集）
 * @param {Array} arr 原数组
 * @returns {Array} 幂集
 */
function generatePowerSet(arr) {
  const result = [[]]; // 初始包含空集（对应用户未选任何规格的场景）
  for (const item of arr) {
    // 给已有子集追加当前项，生成新的子集
    const newSubsets = result.map(subset => [...subset, item]);
    result.push(...newSubsets);
  }
  return result;
}

/**
 * 预处理SKU，生成幂集路径字典
 * @param {Array} skuList 后端返回的完整SKU列表
 * @returns {Map} 幂集路径字典，key为规格路径，value为是否有库存
 */
function createSkuPathMap(skuList) {
  const pathMap = new Map();

  // 只处理有库存的SKU
  skuList.filter(sku => sku.stock > 0).forEach(sku => {
    // 1. 规格转为排序后的键值对数组，保证序列化顺序一致
    const sortedSpecEntries = Object.entries(sku.spec).sort((a, b) => a[0].localeCompare(b[0]));
    // 2. 转为 ['颜色:红', '尺码:S'] 格式的路径片段
    const specPathSegments = sortedSpecEntries.map(([key, value]) => `${key}:${value}`);
    // 3. 生成当前路径的幂集（所有子路径）
    const powerSet = generatePowerSet(specPathSegments);
    // 4. 所有子路径存入Map，标记为有库存
    powerSet.forEach(subset => {
      const pathKey = subset.join('|');
      pathMap.set(pathKey, true);
    });
  });

  return pathMap;
}

/**
 * 判断单个规格选项是否可选
 * @param {Map} pathMap 预处理好的幂集路径字典
 * @param {Object} selectedSpec 用户已选的规格 {颜色: '红', 尺码: 'S'}
 * @param {String} specName 当前要判断的规格名称（如'尺码'）
 * @param {String} specOption 当前要判断的规格选项（如'M'）
 * @returns {Boolean} 是否可选
 */
function isSpecOptionEnabled(pathMap, selectedSpec, specName, specOption) {
  // 合并已选规格和当前判断的选项
  const tempSpec = { ...selectedSpec, [specName]: specOption };
  // 排序、序列化，和预处理逻辑保持一致
  const sortedEntries = Object.entries(tempSpec).sort((a, b) => a[0].localeCompare(b[0]));
  const pathKey = sortedEntries.map(([key, value]) => `${key}:${value}`).join('|');
  // 查字典判断是否存在
  return pathMap.has(pathKey);
}

// ===================== 业务测试示例 =====================
// 模拟后端返回的SKU列表
const skuList = [
  { skuId: '1', spec: { 颜色: '红', 尺码: 'S' }, stock: 10, price: 99 },
  { skuId: '2', spec: { 颜色: '红', 尺码: 'M' }, stock: 5, price: 99 },
  { skuId: '3', spec: { 颜色: '白', 尺码: 'M' }, stock: 0, price: 89 }, // 无库存，会被过滤
  { skuId: '4', spec: { 颜色: '白', 尺码: 'L' }, stock: 20, price: 89 },
];

// 1. 页面初始化时，预处理生成路径字典（仅执行1次）
const skuPathMap = createSkuPathMap(skuList);

// 2. 业务场景测试
// 场景1：用户未选任何规格，判断红色是否可选
console.log(isSpecOptionEnabled(skuPathMap, {}, '颜色', '红')); // true
// 场景2：用户未选任何规格，判断白色是否可选
console.log(isSpecOptionEnabled(skuPathMap, {}, '颜色', '白')); // true（白+L有库存）
// 场景3：用户选了红色，判断S码是否可选
console.log(isSpecOptionEnabled(skuPathMap, { 颜色: '红' }, '尺码', 'S')); // true
// 场景4：用户选了白色，判断M码是否可选
console.log(isSpecOptionEnabled(skuPathMap, { 颜色: '白' }, '尺码', 'M')); // false（白+M无库存）
// 场景5：用户选了白色，判断L码是否可选
console.log(isSpecOptionEnabled(skuPathMap, { 颜色: '白' }, '尺码', 'L')); // true
```

### 5. 幂集算法的核心优势

- **性能极高**：预处理仅在页面初始化时执行 1 次，后续每次判断都是 O (1) 的 Map 查询，不受 SKU 数量影响；
- **逻辑通用**：支持任意数量的规格组，2 组、3 组甚至更多规格，代码逻辑完全不用修改；
- **边界处理完善**：天然支持全选、反选、清空等操作，不会出现传统方案的判断漏洞。

## 四、关键概念区分：幂集 vs 笛卡尔积

很多教程会把两者都叫 “SKU 幂集算法”，但二者的数学本质和业务用途完全不同，避免混淆：

|概念|数学本质|元素数量规律|SKU 业务用途|
|---|---|---|---|
|幂集|单个集合的所有子集|2ⁿ（n 为集合元素个数）|详情页规格选择器的可选状态判断|
|笛卡尔积|多个集合的元素全排列组合|n1×n2×...×nn（n 为每个规格组的选项数）|后台自动生成 SKU 全组合|

---

## 五、注意事项与避坑指南

1. **序列化一致性**：预处理和实时判断的排序、拼接逻辑必须完全一致，否则会出现同一个组合生成不同 key，导致判断错误。
2. **分隔符选择**：拼接路径的分隔符要选不会出现在规格名称 / 选项里的字符（推荐`|`、`^`），不要用逗号、冒号，避免和规格内容冲突。
3. **无库存 SKU 过滤**：预处理时必须过滤掉`stock<=0`的 SKU，否则无库存的组合也会被判断为可选。
4. **性能边界**：幂集的大小是`2ⁿ`，n 为规格组的数量。常规 3 组规格，每个 SKU 的幂集仅 8 个子集，1000 个 SKU 也仅 8000 条数据，完全无压力；但规格组超过 6 个时，数据量会大幅增长，而业务上用户也很难选择超过 6 组的规格，不推荐使用。
5. **空集处理**：幂集中的空集对应 “用户未选任何规格” 的场景，天然符合业务逻辑，无需额外处理。