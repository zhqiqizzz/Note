`rules` 是 **Element UI / Element Plus** 中 `el-form` 组件的核心属性，用于**定义表单字段的验证规则**，配合 `model`（表单数据）和 `el-form-item` 的 `prop`（字段绑定）实现表单验证功能。

## 一、`rules` 的基本结构

`rules` 是一个**对象**，其键名必须与 `el-form.model` 中的字段名（以及 `el-form-item` 的 `prop` 属性）**完全一致**；键值是一个**数组**，包含一条或多条验证规则（支持多规则叠加）。
### 基本格式

```javascript
const rules = {
  // 键名：对应 model 中的字段名
  字段名1: [
    // 规则1
    { 验证属性1: 值1, 验证属性2: 值2, ... },
    // 规则2（可多条）
    { 验证属性3: 值3, ... }
  ],
  字段名2: [
    { ... }
  ]
};
```

## 二、验证规则的常用属性

每条验证规则是一个对象，包含以下常用属性：

|属性名|说明|类型|可选值 / 示例|
|---|---|---|---|
|`required`|是否必填（仅控制验证逻辑，显示星号需用 `el-form-item` 的 `required`）|`boolean`|`true` / `false`|
|`message`|验证失败时的提示信息|`string`|—|
|`trigger`|验证触发方式|`string`|`blur`（失焦）/ `change`（值变化）|
|`pattern`|正则表达式验证（匹配通过则验证成功）|`RegExp`|`/^1[3-9]\d{9}$/`（手机号）|
|`min`|最小长度（字符串）或最小值（数字）|`number`|—|
|`max`|最大长度（字符串）或最大值（数字）|`number`|—|
|`validator`|自定义验证函数（灵活处理复杂验证逻辑）|`function`|`(rule, value, callback) => {}`|
## 三、常见验证类型示例

以下结合 Vue 3 + Element Plus 演示常用验证场景：
### 1. 必填验证

最基础的验证，确保字段不为空。

```javascript
const rules = {
  username: [
    { required: true, message: '请输入用户名', trigger: 'blur' }
  ]
};
```

### 2. 长度验证

控制字符串的长度范围（或数字的大小范围）。

```javascript
const rules = {
  username: [
    { min: 3, max: 10, message: '长度在 3 到 10 个字符', trigger: 'blur' }
  ],
  age: [
    { min: 18, max: 60, message: '年龄在 18 到 60 之间', trigger: 'blur' }
  ]
};
```

### 3. 正则表达式验证

通过正则匹配特定格式（如手机号、邮箱、身份证号等）。

```javascript
const rules = {
  phone: [
    { required: true, message: '请输入手机号', trigger: 'blur' },
    { pattern: /^1[3-9]\d{9}$/, message: '请输入正确的手机号', trigger: 'blur' }
  ],
  email: [
    { required: true, message: '请输入邮箱', trigger: 'blur' },
    { pattern: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/, message: '请输入正确的邮箱', trigger: 'blur' }
  ]
};
```

### 4. 自定义验证函数

处理复杂逻辑（如两次密码输入一致、异步验证等），通过 `validator` 属性定义。
#### 自定义验证函数的格式

```javascript
(rule, value, callback) => {
  // rule：当前验证规则对象
  // value：当前字段的输入值
  // callback：验证完成后的回调函数
  // - 验证通过：callback()
  // - 验证失败：callback(new Error('错误信息'))
}
```

#### 示例：两次密码输入一致

```javascript
const validateConfirmPassword = (rule, value, callback) => {
  if (value === '') {
    callback(new Error('请再次输入密码'));
  } else if (value !== formData.password) {
    callback(new Error('两次输入的密码不一致'));
  } else {
    callback(); // 验证通过
  }
};

const rules = {
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' },
    { min: 6, message: '密码长度不少于 6 位', trigger: 'blur' }
  ],
  confirmPassword: [
    { required: true, message: '请再次输入密码', trigger: 'blur' },
    { validator: validateConfirmPassword, trigger: 'blur' }
  ]
};
```

## 四、全局 `rules` 与局部 `rules`

`rules` 可以定义在两个位置，**局部规则优先级高于全局规则**：
### 1. 全局 `rules`（绑定在 `el-form` 上）

统一管理所有表单项的验证规则，适合大部分场景。

```vue
<template>
  <el-form :model="formData" :rules="formRules">
    <el-form-item label="用户名" prop="username">
      <el-input v-model="formData.username" />
    </el-form-item>
  </el-form>
</template>

<script setup>
const formRules = {
  username: [
    { required: true, message: '请输入用户名', trigger: 'blur' }
  ]
};
</script>
```

### 2. 局部 `rules`（绑定在 `el-form-item` 上）

为单个表单项单独设置规则，覆盖全局规则。

```vue
<template>
  <el-form :model="formData" :rules="formRules">
    <!-- 用户名：使用全局规则 -->
    <el-form-item label="用户名" prop="username">
      <el-input v-model="formData.username" />
    </el-form-item>
    
    <!-- 密码：使用局部规则（覆盖全局） -->
    <el-form-item 
      label="密码" 
      prop="password"
      :rules="[{ required: true, message: '请输入密码', trigger: 'blur' }]"
    >
      <el-input v-model="formData.password" type="password" />
    </el-form-item>
  </el-form>
</template>
```

## 五、完整示例（结合多种验证）

```vue
<template>
  <el-form
    ref="formRef"
    :model="formData"
    :rules="formRules"
    label-width="100px"
    style="max-width: 500px; margin: 50px auto;"
  >
    <!-- 用户名：必填 + 长度 -->
    <el-form-item label="用户名" prop="username">
      <el-input v-model="formData.username" placeholder="请输入用户名" />
    </el-form-item>

    <!-- 手机号：必填 + 正则 -->
    <el-form-item label="手机号" prop="phone">
      <el-input v-model="formData.phone" placeholder="请输入手机号" />
    </el-form-item>

    <!-- 密码：必填 + 长度 -->
    <el-form-item label="密码" prop="password">
      <el-input v-model="formData.password" type="password" placeholder="请输入密码" />
    </el-form-item>

    <!-- 确认密码：必填 + 自定义验证 -->
    <el-form-item label="确认密码" prop="confirmPassword">
      <el-input v-model="formData.confirmPassword" type="password" placeholder="请再次输入密码" />
    </el-form-item>

    <el-form-item>
      <el-button type="primary" @click="submitForm">提交</el-button>
      <el-button @click="resetForm">重置</el-button>
    </el-form-item>
  </el-form>
</template>

<script setup>
import { ref, reactive } from 'vue';
import { ElMessage } from 'element-plus';

const formRef = ref(null);

// 表单数据
const formData = reactive({
  username: '',
  phone: '',
  password: '',
  confirmPassword: ''
});

// 自定义验证：确认密码
const validateConfirmPassword = (rule, value, callback) => {
  if (value === '') {
    callback(new Error('请再次输入密码'));
  } else if (value !== formData.password) {
    callback(new Error('两次输入的密码不一致'));
  } else {
    callback();
  }
};

// 验证规则
const formRules = {
  username: [
    { required: true, message: '请输入用户名', trigger: 'blur' },
    { min: 3, max: 10, message: '长度在 3 到 10 个字符', trigger: 'blur' }
  ],
  phone: [
    { required: true, message: '请输入手机号', trigger: 'blur' },
    { pattern: /^1[3-9]\d{9}$/, message: '请输入正确的手机号', trigger: 'blur' }
  ],
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' },
    { min: 6, message: '密码长度不少于 6 位', trigger: 'blur' }
  ],
  confirmPassword: [
    { required: true, message: '请再次输入密码', trigger: 'blur' },
    { validator: validateConfirmPassword, trigger: 'blur' }
  ]
};

// 提交表单
const submitForm = async () => {
  try {
    await formRef.value.validate();
    ElMessage.success('验证通过，提交成功！');
  } catch {
    ElMessage.error('请完善表单信息！');
  }
};

// 重置表单
const resetForm = () => {
  formRef.value.resetFields();
};
</script>
```

## 六、注意事项

1. **`prop` 必须与 `rules` 键名一致**：否则验证无法绑定到对应字段。
2. **`trigger` 的选择**：
    
    - `blur`：适合输入框、文本域等，用户输入完成失焦后验证；
    - `change`：适合下拉框、单选框、复选框等，值变化时立即验证。
    
3. **自定义验证必须调用 `callback()`**：验证通过时调用 `callback()`，失败时调用 `callback(new Error('错误信息'))`，否则验证会一直处于 “等待” 状态。
4. **`required` 的视觉与逻辑分离**：`el-form-item` 的 `required` 仅显示星号（视觉效果），验证逻辑需在 `rules` 中设置 `required: true`。