# 项目文件概览

本文档概述 `mlsub` 项目中每个源文件的职责、主要内容以及它们之间的协作关系，便于快速了解整体结构。说明按照功能模块划分，覆盖当前工作目录下的全部文件。

## 顶层构建与说明

- `README.md`
  - 提供英文版项目简介，解释 MLsub 类型推断器的目标、依赖（Menhir）与交互示例。
  - 展示若干 OCaml 片段以及推断出的类型，帮助理解类型系统特性（如子类型、Y 组合子）。
- `Makefile`
  - 定义本地构建规则：通过 `ocamlbuild` 生成 `main.byte`，包含调试信息与 Menhir 解析表。
  - 定义 `build-js` 目标，将核心代码编译为 `webpage.byte` 后再由 `js_of_ocaml` 产出浏览器用的 `mlsub.js`。
  - 使用 `_build.js` 目录区分浏览器版本的中间文件。

## Web Demo 前端

- `index.html`
  - 浏览器端 Demo 页面，加载 `mlsub.js`、`style.css`，提供左右双栏：左侧编辑 ML 片段，右侧显示类型推断结果。
  - 默认填充示例代码，涵盖函数、高阶函数、记录与递归对象。
- `style.css`
  - Solarized 风格样式表，控制页面布局、配色，以及类型输出区域的排版。
  - 对 `.binding`、`.name`、`.type` 等类设置字体与伪元素，模拟 OCaml REPL 风格。
- `webpage.ml`
  - `js_of_ocaml` 前端逻辑：在页面加载后读取编辑器内容，调用解析/类型推断，并把结果渲染到 `#output`。
  - 实现 `update` 函数，将输入文本转为模块 AST，再调用 `Typecheck.infer_module` 获取签名。
  - 使用 DOM API 构造类型列表节点，对错误分支显示错误文本。

## 语法前端（解析流程）

- `exp.ml`
  - 抽象语法树定义，所有语法节点都带 `Location.t`（配合错误定位）。
  - 支持表达式、参数/实参、模式匹配、记录、对象、模块项等结构。
- `lexer.mll`
  - 基于 `ocamllex` 的词法分析器。维护括号栈以判断缩进/换行是否重要（Block/Toplevel vs Paren 等），并对关键字、符号、标识符、整数进行分类。
  - 将换行和注释折叠，再结合 `Push/Pop` 动作与 `state` 枚举辅助错误恢复。
- `parser.mly`
  - Menhir 语法文件，参数化的 functor `Parser.Make (L)` 对接定位器。
  - 定义表达式语法、函数参数/关键字参数、模式、类型表达式以及模块声明。
  - 使用 `located`、`mayfail` 等语法糖统一处理可选节点和错误恢复。
  - 输出 `Exp.exp`、`Exp.modlist`、`Typector.typeterm` 等数据结构。
- `source.ml`
  - 将 `Location`、`Lexer`、`Parser` 整合；实现 `ReadSource` functor 负责从 `Location.source` 读取文本、逐 token 交给 Menhir 增量接口。
  - 含错误恢复逻辑：当解析器进入错误态时，跳过不合法 token 并调用回调报告错误。
  - 对外暴露 `parse_modlist`，供命令行工具与前端直接使用。
- `location.ml`
  - 表示源文件位置、行列范围；提供从字符串或文件构建 `Location.source` 的工具。
  - 支持打印出错行、计算是否包含关系、生成 `LocSet` 集合等，方便错误信息关联源位置。
- `error.ml`
  - 统一定义错误类型（如语法错误、作用域冲突、类型冲突原因等）。
  - `Error.print` 根据 `Location` 输出人类可读的诊断信息，包含源码片段和提示。
  - 提供 `Error.Fatal` 异常与辅助构造函数，供解析和类型推断阶段抛出。

## 基础工具组件

- `variance.ml`
  - 构建类型变型运算（`Pos`/`Neg` 极性），提供合并、比较和符号化输出等辅助函数。
- `symbol.ml`
  - 实现字符串驻留：`Symbol.intern` 将字符串映射到唯一整数 ID 并缓存。
  - 保存 FNV-1a 哈希值，供快速字段查找和完美哈希表生成使用。
  - `Symbol.Dictionary` 子模块：根据字段符号生成无碰撞散列表参数，用于对象结构映射。
- `symbol.mli`
  - 暴露符号相关接口与 `Dictionary` 辅助 API，供其它模块创建符号、计算哈希位置等。
- `intmap.ml`
  - 历史上用于高性能整数集合/映射的数据结构实现，文件包含两套注释掉的 Patricia 树方案。
  - 当前有效代码为 `Fake` functor，基于 OCaml `Set` 模拟接口，供类型系统在需要集合操作时使用。
- `intmap_test.ml`
  - 针对旧版 Patricia 树实现的随机测试、性质测试和基准代码。
  - 现阶段主要作为参考示例保留，`Fake` 版本未启用这些测试。

## 类型构造与约束系统

- `typector.mli`
  - 描述类型构造器层的抽象接口：类型变量、具名类型、函数/对象组件、类型别名与不透明类型的注册与展开。
  - 定义 `Components` 模块的操作（合并、比较、映射等），以及构造函数 `ty_fun`、`ty_obj` 等。
- `typector.ml`
  - 实现 `Components` 结构，具体描述函数型、对象型、基类型（含别名/不透明类型）的内部存储。
  - 负责根据极性组合组件、判断子类型冲突、展开别名，并维护类型定义的 stamp/variance 信息。
  - 对函数参数、关键字参数、对象字段等在不同极性下的合并行为进行了细致处理。
- `types.mli`
  - 抽象类型自动机接口，声明 `state`、`dstate`、`cons`、`compile_type`、`constrain` 等关键操作。
  - 提供自动机的确定化、最小化、可达性分析等函数原型。
- `types.ml`
  - 类型自动机的主体实现：维护极性状态图、流约束、组件展开与别名展开策略。
  - 实现 `TypeLat` 子模块，处理类型集合的并/交（在正/负极性下），并将 `Typector` 的组件映射为自动机节点。
  - 提供 `constrain` 实现类型双向约束、`determinise`/`minimise` 进行自动机构建与优化，以及 `decompile_automaton` 用于回译类型表示。

## 类型推断主流程

- `typecheck.ml`
  - 连接 AST 与类型自动机：定义类型方案 `scheme/dscheme`、环境合并、类型合取/析取操作。
  - 包含针对模式匹配的矩阵分割逻辑、关键字参数处理、类型注解检验等核心算法。
  - `infer_module`（在文件后半部）对模块项进行逐项推断，收集错误并返回签名信息，供 CLI 和前端调用。
- `camlgen.ml`
  - 实验性后端：将内部表达式降级为 OCaml AST（通过 `caml_exp`）并格式化输出，便于调试推断结果。
  - 处理对象字段布局、生成辅助 `get_field` 函数等；部分语言特性尚未完成（存在 `assert false` 占位）。
- `main.ml`
  - 命令行入口。读取文件并构建 `Location.source`，调用 `Source.parse_modlist` 和 `Typecheck.infer_module`。
  - 若推断成功则通过 `Typecheck.print_signature` 输出结果；否则使用 `Error.print` 报告错误。
  - 包含若干注释掉的实验代码（REPL、子类型检查、自动机调试等）。

## 其他资源

- `error.ml`（已在解析部分提及）同时供类型推断阶段调用，因此在此额外强调：它集中定义冲突原因枚举，包括函数参数缺失、对象字段缺失、类型不兼容等，用于在 `constrain` 时形成精准报错。
- `index.html` 里引用的 `mlsub.js` 并未包含在仓库中——这是通过 `Makefile` 的 `build-js` 规则生成的产物。

---

若需快速入门，可从 `typecheck.ml` 的 `infer_module` 流程入手，结合 `types.ml` 的自动机实现理解核心算法；再通过 `webpage.ml` 观察前端如何调用这些 OCaml 模块，便能贯穿解析 → 类型构建 → 推断 → 展示的完整链路。
