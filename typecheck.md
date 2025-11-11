# typecheck.ml 函数流程说明

`typecheck.ml` 负责把前端语法树与类型自动机构件结合，完成 MLsub 的推断、约束与模式检查。本节列出文件内所有顶层函数/值与关键算法的执行流程。

## 数据结构与别名
- `scheme`：记录表达式的类型方案，包含 `environment`（变量名映射到负极状态）与 `expr`（表达式正极状态）。
- `dscheme`：`scheme` 的确定化/最小化版本，环境和表达式使用 `Types.dstate`。
- `typing`：模块内使用的高阶别名，表示接收环境映射后返回 `scheme` 的函数。
- `case_matrix`、`pat_split`、`pat_desc` 等：辅助描述模式匹配矩阵、拆分策略与生成的类型约束。

## 函数与值

### `to_dscheme`
1. 收集方案中的所有 `state`（表达式输出与环境条目）。
2. 调用 `Types.determinise` 构造确定化状态并取得 `remap` 映射。
3. 对确定化结果执行 `Types.minimise`。
4. 将环境和表达式全部通过 `remap`→`minimise` 映射成 `dstate`，生成 `dscheme`。

### `clone_scheme`
1. 调用 `Types.clone` 复制状态图，并在回调中使用提供的 `f loc` 函数将 `dscheme` 的环境和表达式重新映射到新状态。
2. 返回全新的 `scheme`，避免直接共享自动机节点。

### `constrain'`
1. 对 `Types.constrain loc p n` 的错误列表逐项调用 `err` 回调。
2. 只要出现错误就把 `success` 标记为 `false`；最终返回是否全部约束成功的布尔值。

### `dump_scheme`
1. 使用 `Types.print_automaton` 打印主要表达式节点与环境变量节点。
2. 仅在调试时调用，帮助可视化自动机状态。

### `constrain`
1. 选项性地在 `debug` 模式下打印约束前后的自动机。
2. 对输入 `(state * state)` 列表逐个调用 `Types.constrain`，将所有错误交给 `err` 处理。
3. 若存在错误则返回 `compile_type ctx Pos (TZero Pos)` 代替原结果，否则返回 `output`。

### `ty_join` / `ty_join_neg`
1. 创建一对互联的 `(neg, pos)` 状态。
2. 将两个操作数分别约束到新节点的负边或正边。
3. 确保无错误后返回正极（`ty_join`）或负极（`ty_join_neg`）结果，用于表达式合并。

### `join_scheme`
1. 使用 `ty_join` 合并两个方案的表达式状态。
2. 环境部分逐键合并：同名变量使用 `ty_join_neg` 取交，否则沿用唯一的那份。

### `ascription`
1. 将目标类型先编译为正极状态 `s`。
2. 准备一个“顶类型”作为参数环境的上界，构造临时 `dscheme`。
3. 借助 `Types.subsumed` 检查表达式与环境是否满足注解要求；失败则抛出异常。
4. 成功时返回仅含表达式状态的新方案（环境清空）。

### `env_join`
1. 以 `SMap.merge` 合并两个环境映射。
2. 同名变量使用 `ty_join_neg` 求交，否则保留出现的那一条。

### `combine_splits`
1. 递归合并左右模式拆分结果，分别处理整数、对象、默认分支。
2. 对对象字段按哈希排序并补齐缺失字段（填入 `_` 通配模式）。
3. 对结果模式返回合并后的分支描述和新的守卫变量类型（通过 `ty_join_neg` 取交）。

### `split_cases`
1. 按当前列的模式种类拆分 `case_matrix`：通配、变量绑定、整数、对象或模式并列。
2. 对对象分支先按字段哈希排序检测重复，再转为 `PSObj`。
3. 调用 `combine_splits` 把当前头部与递归拆分结果合并，返回守卫变量与拆分描述。

### `dump_decision_tree`
1. 调试函数：递归打印 `split_cases` 生成的决策树结构。
2. 区分 `PSAny`、`PSInt`、`PSObj` 等分支并缩进展示。

### `merge_desc` / `merge_desctypes` / `merge_fields`
1. 针对两份模式描述取 meet，过程中检查类型不匹配并抛错。
2. 对函数/对象标签逐项合并，必要时引用默认分支。
3. `merge_fields` 针对同名字段递归调用 `merge_desc`。

### `merge_match_descs`
1. 合并两个 `match_desc`（一列模式描述 + 结果方案）。
2. 逐列调用 `merge_desc`，最终使用 `join_scheme` 组合结果表达式。

### `describe_cases`
1. 基于 `split_cases` 的拆分结果递归构建每列的 `pat_desc`。
2. 对整数、对象分支合并所有分支的描述；必要时懒加载默认分支。
3. 标记被实际使用到的 case；未触发的在主流程中报 `Error.Unused_case`。

### `describe_fields`
1. 辅助函数：将对象字段列表与 `match_desc` 对齐，构造字段到 `pat_desc` 的映射。

### `variables_bound_in_pat`
1. 遍历模式列表，记录变量绑定所在位置。
2. 检查重复绑定与部分分支缺失绑定（`Error.Partially_bound` / `Error.Rebound`）。

### `add_singleton`
1. 为变量创建一对 `(neg, pos)` 新状态。
2. 组装只包含该变量的 `scheme` → `dscheme` 并装入环境。

### `var`
1. 实用函数：若变量在环境中存在，则返回约束对 `[expr_state, env_state]`，否则返回空表。

### `failure`
1. 构造一个“失败”方案：空环境，表达式为 `⊥`，用于语法错误回退。

### `ctx0`
1. 初始化类型构造上下文，引入内建 `int`/`unit`/`bool`/`list` 类型。

### `ty_int` / `ty_unit` / `ty_bool`
1. 基于 `ctx0` 构建返回具体基础类型组件的函数（位置参数是 `Location`）。

### `ty_list`
1. 构造 `list` 类型的命名类型调用，封装成函数供调用方指定元素类型的生成方式。

### `pat_desc_to_type`
1. 将模式描述 `pat_desc` 转换为负极自动机状态。
2. 根据 `pcases` 的不同分支，构造整型或对象类型并通过 `ty_join_neg` 合并。

### `typecheck` / `typecheck'`
1. 外层 `typecheck` 识别 `None`（语法错误）返回 `failure`，其余交给 `typecheck'`。
2. `typecheck'` 对 AST 各构造分别处理：
   - 变量：从环境中克隆方案，若缺失报 `Error.Unbound`。
   - Lambda：逐个参数构建新的环境条目（支持类型注解与默认值），再拼装函数类型。
   - `let`：推断被绑定表达式，将结果 `dscheme` 加入环境再推断主体，环境合并使用 `env_join`。
   - `rec`：为自引用变量提供初始单例，约束函数体满足递归关系。
   - 应用：依次推断函数与实参，建立形参约束、关键字参数集合与返回类型流向。
   - 顺序、条件、常量、列表与匹配等分支依照语义建立相应约束，确保布尔条件、列表结构、字段存在等。其中 `If` 语句的处理流程如下：
     1. 分别调用 `typecheck` 推断条件、then 分支、else 分支，得到 `(envC, exprC)`、`(envT, exprT)`、`(envF, exprF)`。
     2. 通过两次 `env_join` 合并环境：先合并 then/else 分支捕获的自由变量，再与条件表达式的环境合并，形成整句的环境贡献。
     3. 借助 `Types.cons Neg (ty_bool loc)` 构造布尔类型的负极节点，并向 `constrain` 施加 `[exprC, bool_neg]` 约束，确保条件表达式可视为布尔。
     4. 使用 `ty_join exprT exprF` 计算 then/else 结果类型的最小上界，作为整句的表达式节点。
     5. 将结果包装回 `{ environment; expr }` 结构返回，因此 `if` 表达式在类型上体现为“分支类型合并+条件为布尔”的组合约束。
   - 对 `Match`：构造 case 矩阵、计算 `describe_cases`，约束 scrutinee 与每个模式的型态，并标记未使用 case。
   - 对象构造与字段访问：对字段表达式建立约束，生成对象类型或字段提取要求。
3. 返回组合后的 `scheme`，供上层环境或调用者使用。

### `ty_cons` / `ty_fun2`
1. 小型构造辅助：`ty_cons` 将组件函数包装成 `TCons`；`ty_fun2` 构建二元函数类型骨架。

### `ty_polycmp` / `ty_binarith`
1. 预定义函数类型模板：多态比较与整数二元算术。

### `predefined`
1. 默认导入的顶层符号（算术、比较、`error` 等），以 `(name, typeterm)` 列表形式给出。

### `gamma0`
1. 将 `predefined` 转换为 `dscheme` 并注入初始环境映射。

### `infer_module`
1. 递归遍历模块项列表，管理类型上下文与顶层环境：
   - `MType` / `MOpaqueType`：向上下文注册别名或不透明类型。
   - `MDef`：先假设目标函数递归，再用 `typecheck` 函数推断函数体，将 `SDef` 签名加入输出。
   - `MLet`：推断表达式，延续环境时通过 `add_singleton` 为绑定变量提供值级入口。
2. 累计的签名状态列表经 `Types.determinise`、`Types.optimise_flow`、`Types.minimise` 进一步精简。
3. 返回 `(context, signature_items)` 作为模块推断结果。

### `print_signature`
1. 使用 `Types.clone` 获取所有签名条目的自动机状态列表。
2. 调用 `decompile_automaton` 回译成 `typeterm`。
3. 根据 `SLet`/`SDef` 生成 `val`/`def` 行输出。
