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

> TypeScript 的类型只存在于编译阶段，主要用于静态检查、类型提示和提升代码可维护性，运行时并不会保留类型。

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
> TypeScript 可以给变量、函数参数、函数返回值添加类型约束，提前发现类型错误。