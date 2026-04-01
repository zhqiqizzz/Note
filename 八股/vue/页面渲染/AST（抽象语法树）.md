AST 全称 **Abstract Syntax Tree**，即**抽象语法树**，是前端工程化工具（Babel、ESLint、webpack、代码压缩工具等）的核心底层基础。简单来说，它是**把人类可读的源代码，转换成计算机容易处理的「树状数据结构」**，是代码分析、转换、优化的桥梁。
## 一、核心定义

AST 是源代码的**抽象语法结构的树状表示**：

- 树的每个节点，对应代码中的一个语法结构（比如变量声明、函数定义、表达式、运算符等）；
- 它忽略了代码中的非语法细节（比如空格、换行、注释、分号等），只保留代码的核心逻辑结构；
- 是编译器 / 解释器处理代码的**中间产物**—— 先把代码解析成 AST，再基于 AST 做后续的分析、转换、生成代码等操作。

## 二、直观示例：代码 → AST

我们用一段最简单的 JS 代码，直观感受 AST 的结构：
### 源代码

```javascript
const a = 1 + 2;
```

### 对应的 AST 结构（JSON 简化版）

```json
{
  "type": "Program", // 根节点：整个程序
  "body": [
    {
      "type": "VariableDeclaration", // 变量声明节点
      "kind": "const", // 声明类型：const
      "declarations": [
        {
          "type": "VariableDeclarator", // 变量声明符节点
          "id": {
            "type": "Identifier", // 标识符节点：变量名
            "name": "a"
          },
          "init": {
            "type": "BinaryExpression", // 二元表达式节点：1 + 2
            "operator": "+", // 运算符
            "left": {
              "type": "Literal", // 字面量节点：1
              "value": 1
            },
            "right": {
              "type": "Literal", // 字面量节点：2
              "value": 2
            }
          }
        }
      ]
    }
  ]
}
```

### 树状可视化

```plaintext
Program (整个程序)
└── VariableDeclaration (变量声明: const)
    └── VariableDeclarator (声明符)
        ├── Identifier (变量名: a)
        └── BinaryExpression (二元表达式: +)
            ├── Literal (字面量: 1)
            └── Literal (字面量: 2)
```

可以看到，代码的每一个语法单元，都被拆解成了有明确类型的节点，计算机可以通过遍历这棵树，轻松理解代码的逻辑结构。

## 三、AST 的生成过程

把源代码转换成 AST，分为两个核心步骤：

### 1. 词法分析（Lexical Analysis）

也叫**分词**，把源代码字符串，拆分成一个个独立的「Token」（词法单元）。

- Token 是代码的最小语法单元，比如关键字（`const`、`function`）、标识符（变量名、函数名）、运算符（`+`、`=`）、字面量（`1`、`'hello'`）、标点符号（`{`、`}`、`;`）等；
- 会忽略空格、换行、注释等非语法内容。

#### 示例：词法分析结果

```javascript
// 源代码
const a = 1 + 2;

// 拆分后的 Token 流
[
  { type: 'Keyword', value: 'const' },
  { type: 'Identifier', value: 'a' },
  { type: 'Punctuator', value: '=' },
  { type: 'Numeric', value: '1' },
  { type: 'Punctuator', value: '+' },
  { type: 'Numeric', value: '2' },
  { type: 'Punctuator', value: ';' }
]
```

### 2. 语法分析（Syntactic Analysis）

把 Token 流，按照编程语言的语法规则，组装成**树状的 AST**。

- 会检查代码的语法是否符合规范，语法错误会在这一步抛出；
- 不同的编程语言、不同的解析器，生成的 AST 结构会有差异，但核心逻辑一致。

## 四、前端开发中 AST 的核心应用场景

AST 是前端工程化的基石，几乎所有你常用的工具，底层都依赖 AST：

### 1. Babel：ES6+ 转 ES5

Babel 的核心工作流程就是基于 AST：

1. **解析（Parse）**：把 ES6+ 代码解析成 AST；
2. **转换（Transform）**：遍历 AST，把 ES6+ 的语法节点（比如箭头函数、`let/const`、类）转换成 ES5 的语法节点；
3. **生成（Generate）**：把转换后的 AST，重新生成 ES5 代码。

### 2. ESLint：代码语法检查与风格校验

ESLint 通过 AST 分析代码结构：

1. 把代码解析成 AST；
2. 遍历 AST，检查每个节点是否符合规则（比如是否用了`var`、变量名是否符合规范、是否有未使用的变量）；
3. 发现不符合规则的节点，抛出警告或错误。

### 3. webpack：模块打包与依赖分析

webpack 通过 AST 分析模块的依赖关系：

1. 解析入口文件的 AST；
2. 遍历 AST，找到所有 `import`、`require` 语句，识别依赖模块；
3. 递归解析依赖模块的 AST，构建完整的依赖图；
4. 基于依赖图，把所有模块打包成最终的 bundle。

### 4. 代码压缩工具（Terser、UglifyJS）

通过 AST 实现代码压缩和优化：

1. 解析代码成 AST；
2. 遍历 AST，删除无用代码（Tree Shaking）、简化变量名（把长变量名改成短的）、合并表达式；
3. 生成压缩后的代码。

### 5. 代码编辑器的智能功能

VS Code、WebStorm 等编辑器的智能提示、语法高亮、自动补全、代码重构（比如重命名变量），底层都是基于 AST 分析代码结构实现的。

### 6. 自定义代码转换工具

你可以基于 AST，开发自己的代码转换工具，比如：

- 批量把项目中的 `px` 转换成 `rem`；
- 批量给函数添加日志；
- 批量替换项目中的旧 API。

## 五、常用的 JS AST 解析器

前端开发中，常用的 JS/TS AST 解析器有：

|解析器|特点|代表应用|
|:--|:--|:--|
|**Acorn**|轻量、高性能、符合 ESTree 规范|webpack、Rollup|
|**@babel/parser**|Babel 官方解析器，支持最新 ES 提案、TypeScript、JSX|Babel|
|**Espree**|ESLint 官方解析器，基于 Acorn|ESLint|
|**TypeScript Compiler API**|TypeScript 官方编译器，支持 TS 语法解析|TypeScript 编译、TS-ESLint|
## 六、实战：用 Acorn 生成 AST

我们用 `acorn` 库，写一个最简单的示例，亲手生成 AST：
### 1. 安装依赖

```bash
npm install acorn
```

### 2. 代码示例

```javascript
const acorn = require('acorn');

// 源代码
const code = `const a = 1 + 2;`;

// 解析成 AST
const ast = acorn.parse(code, {
  ecmaVersion: 'latest' // 指定 ES 版本
});

// 打印 AST
console.log(JSON.stringify(ast, null, 2));
```

### 3. 在线可视化工具

如果你想更直观地看 AST，可以用在线工具：

- **AST Explorer**：[https://astexplorer.net/](https://astexplorer.net/)
    
    可以选择不同的解析器，输入代码，实时查看对应的 AST 结构，还能看到 Babel 转换前后的 AST 对比，是学习 AST 的最佳工具。

## 七、总结

AST 是前端工程化的底层核心，它把人类可读的代码，转换成计算机可处理的树状结构，让代码分析、转换、优化成为可能。

- 对于普通开发者，理解 AST 能帮助你更好地使用 Babel、ESLint 等工具，排查工具相关的问题；
- 对于高级开发者，掌握 AST 能让你开发自定义的代码转换工具，提升开发效率。