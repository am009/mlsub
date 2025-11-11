# 从解析到类型推断的流程概览

本文梳理 MLsub 工具链从读取源码到输出类型签名的关键步骤，覆盖词法、语法、AST 构建、类型推断与签名打印。括号内引用为相关函数所在文件及行号，便于交叉查阅。

## 1. 入口：读取源文件
- 命令行程序遍历传入的每个文件，依次执行 `process`（`main.ml:100-108`）。
- `Location.of_file` 将文件内容与行列信息打包成 `Location.source`，以便后续错误定位。
- 若整个流程未记录错误标记，则调用 `print_signature` 输出推断结果。

## 2. 词法分析
- 解析管线由 `Source.parse_modlist` 驱动（`source.ml:100-102`），其内部构建 `ReadSource` functor 来封装词法与语法组件。
- 词法器由 `Lexer.Make` 生成（`lexer.mll:4-25`），核心入口是 `lex`（`lexer.mll:41-92`）：
  - 根据当前嵌套状态识别换行是否重要，维护显式的括号/块栈（`lexer.mll:27-30`）。
  - 将关键字、标识符、整数字面量等转换成 Menhir token，必要时记录 `Push/Pop` 动作帮助错误恢复。
  - 调用失败时抛出 `Bad_token`，由 `ReadSource.read_token` 捕获并转化为 `Error.Fatal`（`source.ml:32-38`）。

## 3. 语法分析与 AST 生成
- `Parser.Make` 根据 `parser.mly` 生成增量解析器，`ReadSource` 通过 `MenhirInterpreter` 驱动状态机（`source.ml:69-88`）。
- 自定义的 `located` 宏在语法动作中附加源区间（`parser.mly:45-52`），使得所有 AST 节点携带 `Location.t`。
- 顶层规则 `modlist` 解析为 `Exp.modlist`（`parser.mly:58-67`），其中 `Exp` 模块定义了表达式、模式、模块项等数据类型。
- 错误恢复：当解析器进入 `HandlingError`，`ReadSource` 会调用 `skip_bad_tokens` 丢弃输入并报告错误（`source.ml:78-88`）。

## 4. 模块级推断调度
- 解析输出经过 `infer_module`（`typecheck.ml:758-785`），这是类型推断的总入口。
  - 该函数迭代模块项，维护两份状态：类型上下文 `tyctx`（用于类型定义）与值环境 `gamma`（记录值绑定的类型方案）。
  - `infer` 内部维护三元组 `(ctxM, envM, sigM)`，分别对应：
    - `ctxM`：类型上下文（`Typector.context`），记录当前模块已声明的类型别名/不透明类型。每处理一个 `type` 项就调用 `Typector.add_type_alias` 或 `add_opaque_type` 更新它。
    - `envM`：值环境（`Symbol.Map`），追踪尚未消费完的顶层绑定所贡献的局部约束，用于跨顶层项保持一致性。处理完某项后会用 `env_join` 合并新约束，并在 `let` 情形中移除已出作用域的变量（`typecheck.ml:773-778`）。
    - `sigM`：累积的签名条目列表，元素形如 `SDef/SLet` 携带推断出的表达式状态。遍历完成后，`sigM` 中的所有状态将进入确定化/最小化流程。
  - 具体处理逻辑：
    - `MType`/`MOpaqueType` 调用 `Typector.add_type_alias`/`add_opaque_type` 扩展 `ctxM`（`typector.ml:275-327`）。
    - `def` 项使用 `typecheck'` 推断函数体，并借助 `Rec` 包装允许递归引用；更新后的方案塞入 `sigM`（`typecheck.ml:766-771`）。
    - `let` 绑定先推断右侧表达式，再通过 `add_singleton` 将变量加入环境，最终把约束结果封装成 `SLet` 追加到 `sigM`（`typecheck.ml:772-778`）。

### `infer` 递归逻辑解析
`infer`（`typecheck.ml:759-778`）作为 `infer_module` 的内部递归函数，按照模块项构建最终签名：
- **递归终止**：遇到空列表返回 `(tyctx, SMap.empty, [])`，使递归收束并向上传递累计的环境/签名。
- **类型定义分支**：
  1. 调用 `add_type_alias` 或 `add_opaque_type` 更新上下文 `tyctx`。
  2. 递归处理后继模块项，保持值环境与签名未变化（类型声明不引入新值）。
- **函数定义 (`MDef`)**：
  1. 使用 `typecheck'` 推断递归函数体（通过 `Rec` 包装允许自引用）。
  2. 将生成的方案暂存于环境 `gamma` 用于后续项。
  3. 将函数表达式状态 `exprF` 封装为 `SDef` 并插入 `sigM`。
  4. 返回时用 `env_join` 合并当前函数环境与递归结果 `envM`。
- **值绑定 (`MLet`)**：
  1. 推断右侧表达式 `exprE`。
  2. 递归处理剩余项时向环境添加单例约束，确保变量在后续可用。
  3. 结果阶段移除该变量（因为顶层 `let` 定义不会向输出环境泄漏），同时把收集到的约束封装为 `SLet`。

整个递归保障了：类型信息在 `ctxM` 中累积、值级约束在 `envM` 中安全地传递与清理、输出签名按出现顺序记录于 `sigM`。

## 5. 表达式类型推断
`typecheck` 是外层调度函数，负责对 `Exp.exp`（带 `Location.t` 的可选节点）进行处理：
- 若节点为 `None`（解析阶段已记录错误），直接返回 `failure`（`typecheck.ml:502-505`）。
- 否则调用 `typecheck' ctx err gamma loc exp`，进入模式匹配驱动的主要分支（`typecheck.ml:506-715`）。

`typecheck'` 对每种表达式构造分别执行以下步骤：
- **变量 (`Var`)**：从环境 `gamma` 查找方案并使用 `clone_scheme` 克隆（确保状态唯一），找不到则通过错误回调报告 `Error.Unbound` 并返回 `failure`（`typecheck.ml:507-510`）。
- **函数 (`Lambda`)**：
  1. `check_params` 递归处理参数（`typecheck.ml:512-539`），对带注解的参数先用 `Types.compile_type_pair` 构建 `(neg,pos)` 状态并更新环境。
  2. 默认参数在体内增加额外约束：限定默认值类型兼容形参（`typecheck.ml:531-539`）。
  3. `build_funtype` 根据生成的环境拼装 `ty_fun` 组件并用 `Types.cons` 包装（`typecheck.ml:541-557`）。
- **`let`/`rec`**：
  - `Let` 先推断右侧表达式，再将 `dscheme` 写入环境推断主体，环境合并使用 `env_join`（`typecheck.ml:558-566`）。
  - `Rec` 建立新流对 `(recN, recP)`，在体内强制自引用协调，最后移除递归变量（`typecheck.ml:568-575`）。
- **函数调用 (`App`)**：
  - 先推断函数本身，随后 `check_args` 以左折叠计算位置参数和关键字参数，针对每个实参创建 `(argN,argP)` 风筝并加入约束列表，最终构建新的 `ty_fun` 约束来约束函数类型，返回结果状态（`typecheck.ml:577-612`）。
- **顺序与条件 (`Seq`/`If`)**：
  - `Seq` 强制第一个表达式具有 `unit` 类型并返回第二个的结果（`typecheck.ml:614-622`）。
  - `If` 分别推断条件、then、else，利用 `Types.cons Neg (ty_bool)` 限制条件为布尔，并通过 `ty_join` 合并两个分支的输出类型，环境使用两次 `env_join` 合并（`typecheck.ml:634-642`）。
- **集合与对象构造**：
  - `Nil`/`Cons` 通过新建流对约束元素与尾部列表类型，最终构造 `ty_list`（`typecheck.ml:643-651`）。
  - `Object` 遍历字段，累积环境并利用 `Typector.ty_obj_tag` 创建对象组件（`typecheck.ml:702-707`）。
  - `GetField` 强制对象具有指定字段，返回字段流出的正极状态（`typecheck.ml:708-715`）。
- **模式匹配 (`Match`)**：
  1. 对每个被匹配表达式调用 `typecheck` 并合并环境。
  2. 逐个 case 计算绑定变量、推断 case 体，构造 `case_matrix`（`typecheck.ml:666-685`）。
  3. `describe_cases` 推导守卫信息并标记未使用 case；最后用 `Types.constrain` 将 scrutinee 与守卫类型关联（`typecheck.ml:686-700`）。
- **常量与注解**：
  - `Typed` 直接编译类型并调用 `constrain'` 校验（`typecheck.ml:623-633`）。
  - `Unit`/`Int`/`Bool` 分别用 `Types.cons` + 对应的基础类型构造正极状态（`typecheck.ml:643-640` 前若干行）。

所有分支最终返回 `{ environment; expr }` 结构（`scheme`），供上一层使用。
- 推断过程中所有约束都交由 `Types.constrain` 处理，它会在自动机层面执行子类型检查（`types.ml:170-212`）。

## 6. 类型自动机与别名展开
- `Types.compile_type` 将 `typeterm` 编译为自动机状态（`types.ml:120-154`），内部使用 `TypeLat` 以延迟展开别名。
- 约束合并时，如遇命名类型，会依据 `typedef = TAlias/TOpaque` 选择展开与否（`typector.ml:300-327`、`types.ml:94-118`）。
- 推断结束后，`infer_module` 收集所有输出状态并执行：
  1. `Types.determinise`（`types.ml:234-273`）构造确定化自动机；
  2. `Types.optimise_flow` 精简流边（`types.ml:334-362`）；
  3. `Types.minimise` 合并等价状态（`types.ml:274-333`）。

## 7. 签名生成与打印
- 处理完的签名项携带 `dstate`，`print_signature` 通过 `Types.clone` 和 `decompile_automaton` 回译为 `typeterm`（`typecheck.ml:787-796`、`types.ml:176-233`）。
- 最终，`print_typeterm` 结合上下文将类型以用户可读形式输出（`typector.ml:186-237`），对函数、对象等结构进行漂亮打印。

---

综上，管线将源文件逐步转化为带定位信息的 AST，再通过类型自动机执行约束和简化，最后生成精简且可读的类型签名。每个阶段都在相关模块中有明确的接口，便于扩展与调试。
