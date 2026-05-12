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

这是非常关键的一句话