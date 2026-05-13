# 一、先建立 TypeScript 的核心认知

TypeScript 可以理解为：

> JavaScript + 类型系统 + 编译期检查

它不会改变 JavaScript 的运行逻辑
```TypeScript
const age: number = 18
```

编译后还是普通 JS：
```TypeScript
const age = 18
```

所以面试里可以这样说：

> TypeScript 的类型只存在于编译阶段，主要用于静态检查、类型提示和提升代码可维护性，运行时并不会保留类型

# 二、你最先要掌握的 10 个核心点

## 1. 基础类型

```TypeScript
let name: string = 'Tom'
let age: number = 18
let isLogin: boolean = true
let list: number[] = [1, 2, 3]
let names: Array<string> = ['a', 'b']
```

函数：
```TypeScript
function add(a: number, b: number): number {
  return a + b
}
```

箭头函数：
```TypeScript
const add = (a: number, b: number): number => {
  return a + b
}
```

面试表达：
> TypeScript 可以给变量、函数参数、函数返回值添加类型约束，提前发现类型错误

## 2. 类型推断

这点很重要，不是所有地方都要手写类型
```TypeScript
const count = 1
```

TypeScript 会自动推断：
```TypeScript
const count: 1
```

或者：
```TypeScript
let count = 1
```

推断为：
```TypeScript
let count: number
```

所以实际开发里不一定写成：
```TypeScript
let count: number = 1
```
因为 TS 自己能推断

更推荐：
```TypeScript
const count = 1
const title = 'hello'
const isShow = true
```
只有复杂对象、函数参数、接口数据、组件 props 才重点标类型

## 3. `any`、`unknown`、`never`

这是面试高频
### `any`

```TypeScript
let data: any = 123
data = 'hello'
data = true
data.foo.bar()
```
`any` 等于关闭类型检查，面试表达：

> any 会绕过 TypeScript 的类型检查，失去类型保护能力，所以企业项目里应该尽量少用。

---

### `unknown`

TypeScript

```
let data: unknown = 'hello'
```

`unknown` 表示不知道类型，但比 `any` 安全。

不能直接使用：

TypeScript

```
data.toUpperCase() // 报错
```

必须先判断：

TypeScript

```
if (typeof data === 'string') {  data.toUpperCase()}
```

面试表达：

> unknown 是类型安全的 any。它要求使用前先进行类型收窄。