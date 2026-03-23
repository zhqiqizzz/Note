`el-form` 是 **Element UI / Element Plus** 中构建表单的标准化组件，采用**三层嵌套结构**设计，每层职责明确，协作完成表单的布局、数据绑定和验证功能。

## 一、整体组成结构

`el-form` 由外到内分为三层，缺一不可：

```plaintext
1. el-form（表单容器层）
   └── 2. el-form-item（表单项层）
         └── 3. 表单控件层（如 el-input、el-select 等）
```

## 二、各层详细说明

### 1. 最外层：`el-form`（表单容器）

`el-form` 是整个表单的**根容器**，负责**全局管理**：

- 绑定表单数据对象（`model`）；
- 绑定全局验证规则（`rules`）；
- 控制表单整体布局（标签位置、行内模式等）；
- 提供全局方法（验证、重置等）。

#### 核心属性

|属性名|说明|
|---|---|
|`model`|**必填**，表单数据对象（所有表单字段都需绑定在此对象上）|
|`rules`|全局验证规则对象（与 `model` 字段一一对应）|
|`label-position`|标签位置（`top`/`left`/`right`，默认 `right`）|
|`label-width`|标签宽度（如 `'100px'`，统一控制所有表单项的标签宽度）|
|`inline`|是否为行内表单（表单项横向排列，默认 `false`）|
|`disabled`|是否禁用整个表单（所有表单项禁用，默认 `false`）|
#### 核心方法

通过 `ref` 获取 `el-form` 实例后调用：

- `validate()`：验证整个表单；
- `resetFields()`：重置表单到初始值并清除验证状态；
- `clearValidate()`：清除验证状态（不重置值）。

### 2. 中间层：`el-form-item`（表单项）

`el-form-item` 是**连接容器和控件的桥梁**，负责**单个表单项的管理**：

- 显示表单项标签（`label`）；
- 绑定单个字段的验证（通过 `prop` 关联 `model` 和 `rules`）；
- 显示验证错误信息；
- 控制单个表单项的布局（如单独设置标签宽度）。
#### 核心属性

|属性名|说明|
|---|---|
|`label`|表单项标签文本（如 “用户名”“密码”）|
|`prop`|**验证必填**，绑定 `model` 中的字段名（需与 `rules` 键名一致）|
|`required`|是否显示必填星号（仅视觉效果，验证需通过 `rules`）|
|`rules`|单个表单项的验证规则（优先级高于 `el-form` 的 `rules`）|
|`error`|手动设置错误信息（覆盖自动验证）|
|`label-width`|单个表单项的标签宽度（优先级高于 `el-form` 的设置）|
#### 关键作用：`prop` 属性

`prop` 是 `el-form-item` 最核心的属性，它的作用是**建立 “表单控件 ↔ model 字段 ↔ 验证规则” 的关联**：

- `prop` 的值必须与 `el-form.model` 中的字段名完全一致；
- 只有设置了 `prop`，该表单项的验证才会生效。

### 3. 最内层：表单控件（输入层）

表单控件是**用户直接交互的输入组件**，负责**数据采集**，通过 `v-model` 绑定到 `el-form.model` 的对应字段上。
#### 常见表单控件

|控件名|说明|典型场景|
|---|---|---|
|`el-input`|输入框（文本 / 密码 / 数字）|用户名、密码、手机号|
|`el-select`|下拉选择器|性别、地区、状态选择|
|`el-radio-group`|单选框组|性别、是否同意协议|
|`el-checkbox-group`|复选框组|兴趣爱好、权限选择|
|`el-date-picker`|日期选择器|生日、预约时间|
|`el-switch`|开关|是否启用、状态切换|

#### 关键绑定：`v-model`

表单控件通过 `v-model` 双向绑定到 `el-form.model` 的字段上，实现数据的实时同步：

```vue
<!-- 表单控件通过 v-model 绑定到 formData.username -->
<el-input v-model="formData.username" />
```

## 三、完整组成示例

以下是一个典型的登录表单，清晰展示了三层结构的协作：

```vue
<template>
  <!-- 第一层：el-form 容器 -->
  <el-form
    ref="formRef"
    :model="formData"    <!-- 绑定表单数据对象 -->
    :rules="formRules"  <!-- 绑定全局验证规则 -->
    label-width="80px"   <!-- 统一设置标签宽度 -->
    style="max-width: 400px; margin: 50px auto;"
  >

    <!-- 第二层：el-form-item 表单项（用户名） -->
    <el-form-item label="用户名" prop="username">
      <!-- 第三层：el-input 表单控件 -->
      <el-input v-model="formData.username" placeholder="请输入用户名" />
    </el-form-item>

    <!-- 第二层：el-form-item 表单项（密码） -->
    <el-form-item label="密码" prop="password">
      <!-- 第三层：el-input 表单控件（密码类型） -->
      <el-input v-model="formData.password" type="password" placeholder="请输入密码" />
    </el-form-item>

    <!-- 第二层：el-form-item 表单项（按钮） -->
    <el-form-item>
      <el-button type="primary" @click="submitForm">提交</el-button>
      <el-button @click="resetForm">重置</el-button>
    </el-form-item>

  </el-form>
</template>

<script setup>
import { ref, reactive } from 'vue';

// 第一层：el-form 绑定的 model 数据对象
const formData = reactive({
  username: '', // 对应 el-form-item 的 prop="username"
  password: ''  // 对应 el-form-item 的 prop="password"
});

// 第一层：el-form 绑定的 rules 验证规则
const formRules = {
  username: [ // 键名对应 formData.username 和 prop="username"
    { required: true, message: '请输入用户名', trigger: 'blur' }
  ],
  password: [ // 键名对应 formData.password 和 prop="password"
    { required: true, message: '请输入密码', trigger: 'blur' }
  ]
};

// 获取 el-form 实例，调用全局方法
const formRef = ref(null);
const submitForm = () => formRef.value.validate();
const resetForm = () => formRef.value.resetFields();
</script>
```

## 四、三层协作关系总结

|层级|职责|核心关联点|
|---|---|---|
|`el-form`|全局数据、验证、布局管理|`model`（数据）、`rules`（规则）|
|`el-form-item`|单个表单项标签、验证绑定|`prop`（关联 model 字段和 rules）|
|表单控件|用户输入、数据采集|`v-model`（绑定到 model 字段）|

通过这三层结构，`el-form` 实现了 “数据绑定 → 验证 → 提交” 的完整闭环，是 Element 生态中最常用的组件之一。