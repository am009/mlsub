# parser.mly 语法说明

`parser.mly` 使用 Menhir 描述 MLsub 子语言的抽象语法，配合 `lexer.mll` 生成带位置信息的解析器。本说明按文件结构解读 token、优先级、语法宏与各非终结符的含义。

## Token 定义与优先级
- `%token` 列举所有词法符号：带参数的标识符 `IDENT`、整数 `INT`，以及关键字/操作符（`LET`、`IF`、`ARROW` 等）。
- 优先级/结合性通过 `%right`、`%left`、`%nonassoc` 指定，顺序自上而下决定从弱到强组合：
  - 赋值/箭头/逻辑运算的右结合；
  - 比较运算的不可结合；
  - 加减号左结合；
  - 列表构造 `CONS` 最右结。

## 头部 OCaml 代码与参数
```ocaml
%{
  open Variance
  open Typector
  open Types 
  open Typecheck
  open Exp
  open Location
%}
```
编译后的解析器需要访问这些模块中的类型/构造函数。  
`%parameter <L : Location.Locator>` 使解析器成为带定位器 functor，`Lexer.Make` 会传入具体实现。

## 起始符号
- `%start <Exp.exp> prog`：单个表达式。
- `%start <Exp.modlist> modlist`：模块顶层项列表。
- `%start <Typector.typeterm> onlytype`：单个类型。
- `%start <Typector.typeterm * Typector.typeterm> subsumption`：类型子关系对，用于测试。

## 通用语法宏
- `located(X)`：封装子树，附带 `Location.t` 起止位置。
- `mayfail(X)` / `nofail(X)`：将产生式包装为 `Some` 或 `None`，配合错误恢复。
- `onl` / `snl`：消解换行与可选分号，用于 block 语句。
- `sep_list(Sep, Item)`：通用分隔符列表宏，支持尾随分隔符与错误跳过。

## 顶层结构
- `modlist`：跳过前导空行后，解析 `located(moditem)` 列表，末尾强制 `EOF`。
- `moditem`：四种模块项：
  1. `let` 绑定（值定义）。
  2. `def` 函数（具参数列表与函数体）。
  3. `type ... = ...` 别名。
  4. `type ...`（无右侧）不透明类型。
- `funbody`/`funbody_code`：函数体既可使用 `=` 表达式，亦可 `do ... end` 块，也支持尾随 `: type` 注解。

## 块与表达式
- `block` / `block_exp`：处理 `do ... end` 内的多表达式序列；支持嵌套 `let` 与自动把结尾 `;` 转为 `Unit`。
- `lambda_exp`：函数表达式入口，允许 `<params> e`（匿名函数）、`do ... end` 或普通简单表达式。
- `params` / `paramtype` / `param`：解析位置参数、必选/可选关键字以及可选类型注解。
- `argument`：函数调用时的参数，自动把裸的 `x=` 重写为向 `Var x` 的引用。

## 简单表达式与操作符
- `simple_exp_r`：处理二元运算（通过 `binop` 转为 `App`），列表 `::`、以及 `term`。
- `simple_exp`：调用 `located(nofail(simple_exp_r))`，保证返回带位置信息的表达式。
- `binop`：把 `=`, `<`, `+` 等运算符映射到对应的函数名字符串（如 `"(+)"`）。

## Term 层级
- `term_r` 包含大多数构造：
  - `if ... then ... else ... end`
  - `match`（支持多 scrutinee 列表和 case 列表）
  - 变量、括号、类型注解、函数调用
  - 单元/列表/对象字面量、字段访问、整数、布尔常量
- `term`：再次包裹 `located(nofail(term_r))`。

## 对象字段与模式
- `objfield(body)` / `objfield_pat(body)`：辅助宏，处理对象字面量和模式中的字段绑定（缺省值为同名变量/模式）。
- `nonemptylist_r` / `nonemptylist`：列表字面量语法，构造成 `Cons`/`Nil` 树。

## 模式语法
- `pat_r`：通配符、变量、对象、并列模式、括号与整数字面量。
- `pat`：带位置包装。
- `case`：`CASE patterns -> expr` 结构；`case_r` 表示逗号分隔的模式列表。

## 其他起始语法
- `subsumption`：读取 `t1 <: t2 EOF`，供命令行测试子类型关系。
- `onlytype`：单个 `typeterm`。

## 类型语法
- `variance`：解析 `+`, `-`, `+-`, `-+`。
- `typearg`：处理类型参数的显式变型或默认值。
- `typeparam`：类型构造器参数列表，允许 `_` 占位或显式符号+变型。
- `objtype`：对象字段类型 `(label : type)`。
- `funtype`：函数类型参数，使用递归定义并保留关闭括号以避免 LALR 冲突。
- `typeterm`：综合类型表达式：
  - 具名类型（带可选参数列表）。
  - 函数类型 `(...) -> ...`。
  - 内建 `any` / `nothing`。
  - 对象类型 `{ ... }`。
  - 并/交 `t1 AND/OR t2`（通过 `meetjoin` 转换成 `Variance.Neg/Pos`）。
  - 递归类型 `rec a = t`。
  - 通配符 `_` 与括号表达式。
- `meetjoin`：将 `AND` 映射为 `Variance.Neg`（交），`OR` 映射为 `Variance.Pos`（并）。

## 语义动作约定
多数产生式通过调用 `Exp`, `Typector` 等模块的构造函数来生成 AST，与 `typecheck.ml` 所期望的结构保持一致。  
`located` 宏确保所有带 `Location.t` 的节点在解析阶段就携带位置信息，方便后续错误报告。
