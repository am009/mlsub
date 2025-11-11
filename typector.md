# typector.ml 函数流程说明

`typector.ml` 提供类型构造器层：描述函数/对象/具名类型组件结构，维护类型上下文，并支持打印、别名展开等操作。下面依次说明文件中的主要类型与函数。

## 组件与辅助类型
- `Components` 模块：
  - `locations`：聚合组件内所有 `Location`，用于错误信息。
  - `change_locations`：深度替换组件内的位置信息。
  - `pmap`：按极性映射所有子组件（函数位置翻转极性、对象字段保持极性等）。
  - `pfold`：极性敏感的折叠，用于遍历组件子项。
  - `cmp_component`：判断两个组件是否属于同一“形状”（函数参数个数、对象或同一基类型）。
  - `join`：根据极性执行上界/下界合并；处理函数参数、关键字参数、对象标签等结构化组合。
  - `lte`：判断一个组件是否可赋值给另一个组件，返回可能的冲突列表。
  - `lte'`：`lte` 的布尔包装器。
  - `list_fields`：列出所有字段/参数（用于打印或图遍历）。

## 类型上下文与注册
- `stamp` / `context`：类型别名和不透明类型在上下文中使用的编号系统。
- `builtin_stamp` / `empty_context`：内建 stamp = 0，空上下文从 1 起分配。
- `add_to_context err ctx name ty`
  1. 检查是否重复命名。
  2. 分配新 stamp，把别名/不透明类型写入 `by_name` 和 `by_stamp` 映射。
- `find_by_name ctx name`
  - 若存在记录，返回 `(stamp, variance_list)`，否则 `None`。
- `name_of_stamp ctx stamp`
  - 通过 `by_stamp` 查找原始符号，找不到则报错。

## 打印工具
- `comma`
  - 打印 `,@ ` 分隔符，用于 Format 列表。
- `printp paren pr ppf`
  - 根据 `paren` 决定是否输出括号，包装 `Format` block 调用。
- `print_comp pb ctx pr`
  1. 根据组件类型（函数/对象/具名类型）格式化输出。
  2. 函数组合位置参数、关键字参数与返回类型；对象按标签输出 `{field = ...}`；基类型根据参数 variance 打印符号。
- `needs_paren`
  - 判断 `typeterm` 在当前上下文下是否需要括号。
- `print_typeterm ctx pb`
  1. 根据类型构造输出 OCaml 风格表示（`any`/`nothing`/命名类型/函数/对象/并交/递归等）。
  2. 内部借助 `print_comp` 与 `print_sum` 处理函数和并交类型。
- `print_sum`
  - 在同一极性下递归打印 `TAdd` 链。

## `typeterm` 构造器语义
- `TZero of Variance.polarity`
  - 表示极性相关的零元素：`TZero Pos` 对应“nothing”（上界为空集），`TZero Neg` 对应“any”（下界为全集）。
  - 在 `print_typeterm` 中渲染为 `nothing` / `any`，在类型运算里作为并/交的单位元。
- `TNamed of tyvar * typeterm tyargterm list`
  - 命名类型引用，可携带一组实参。
  - 解析阶段允许参数带显式变型（`VarSpec`）或推断值（`VarUnspec`），随后在 `ty_named` 中根据上下文对齐至 `APos`/`ANeg`/`ANegPos`/`ANone`。
- `TCons of typeterm Components.t`
  - 直接存放函数/对象/基类型组件，绕过命名查找。
  - 常由 `ty_fun`、`ty_obj_l`、`ty_base` 等辅助函数生成，表示已经展开到组件级别的类型结构。
- `TAdd of Variance.polarity * typeterm * typeterm`
  - 广义和/交，极性由 `Variance.Pos`（并，`|`）或 `Variance.Neg`（交，`&`）指示。
  - `print_sum` 与 `ty_add` 负责将其线性化，并在类型 lattice 中合并同极性树。
- `TRec of tyvar * typeterm`
  - 递归类型定义 `rec a = ...`，解析时保持 AST 形态，后续在 `compile_set` / `expand_tybody` 中展开并检查守护性。
- `TWildcard`
  - `_` 通配符，在类型注解或别名中表示推迟求值的占位符。
  - 在 `ty_named` 处理变型时可替换外推入的 `ANegPos` 成对未指定参数。

## 类型构造函数
- `ty_fun pos kwargs res loc`
  - 构造函数组件：从位置参数列表、关键字参数映射与返回类型构建 `Components.Func`。
- `ty_obj_cases tag def loc`
  - 构造对象联合类型（按标签拆分）。
- `ty_obj_tag` / `ty_obj`
  - 构造具名或匿名对象组件；`ty_obj` 相当于匿名 case。
- `ty_obj_l`
  - 从字段列表构造对象组件（便于记录字面量）。
- `ty_base ctx s args loc`
  - 通过 stamp 获取对应 typedef，并结合实参返回 `Components.Base`。
- `ty_named ctx name args loc`
  1. 根据上下文查找具名类型；对照 variance 列表检查实参规格。
  2. 将输入 `tyargterm` 归约为 `APos`/`ANeg`/`ANegPos`/`ANone` 组合并调用 `ty_base`。
- `ty_named' ctx name args loc`
  - 跳过 variance 检查，直接通过 stamp 构造（供内部使用）。

## 上下文扩展
- `add_type_alias err name param_list term ctx`
  1. 建立参数索引表，检测重名。
  2. 通过递归 `check` 推断别名体内各参数的实际极性使用，验证显式注解。
  3. 将推断好的 `(variance, Symbol option)` 数组和别名体 `Components.tybody` 注册到上下文。
- `add_opaque_type err name param_list ctx`
  1. 检查参数列表（必须提供显式 variance）并记录符号。
  2. 使用 `add_to_context` 注册 `TOpaque`。

## 别名展开与 stamp
- `get_stamp`
  - 若组件为 `Base` 且 typedef 是 `TAlias`，返回其 stamp，否则视为内建（`builtin_stamp`）。
- `expand args pol body`
  1. 别名展开辅助函数：根据调用极性从 `args` 数组挑选正向或负向版本。
  2. 对 `BCons` 节点递归调用 `pmap`。
- `expand_alias`
  - 针对 `Components.Base` 且携带 `TAlias` 的组件，展开为底层 `Components` 结构；若遇到 `BParam` 说明不具守护性则报错。
