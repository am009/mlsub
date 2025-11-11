# types.ml 函数流程说明

`types.ml` 实现类型自动机：通过有限状态表示类型集合，支持构造、约束、确定化与最小化。以下按功能分组列出文件中所有顶层数据结构与函数的职责。

## 核心数据结构

- `TypeLat.t`（`types.ml:9-60`）  
  表示单个极性下的类型格节点，分为三种形式：
  - `LZero`：空集/单位元；任何 join 与它结合均保持原值。
  - `LUnexpanded`：包装尚未展开的别名组件，维持惰性，以便后续按需根据 stamp 展开。
  - `LExpanded`：保存已展开的基础组件列表（非空），对应某个极性的多个备选分支。  
  该结构提供 `lift`、`pmap`、`pfold` 等组合函数，以便在不失去惰性的前提下映射、遍历或合并组件。

- `State.state`（`types.ml:74-90`）  
  自动机中的节点记录：
  - `id`：全局唯一编号（由 `fresh_id` 生成），便于比较和哈希。
  - `pol`：节点极性（正/负）。
  - `cons`：一个 `TypeLat.t`，描述该状态在当前极性下的结构约束（函数域、对象字段等）。
  - `flow`：`StateSet.t`，记录与其互为流约束的对端节点（正连到负，反之亦然）。

- `StateSet`（`types.ml:91-94`）  
  通过 `Intmap.Fake` 生成的集合结构，以 `State.state.id` 为比较键，支持集合运算、遍历与 fold。

- `StateTbl` / `VarMap` 等哈希/映射（`types.ml:96-107`）  
  `StateTbl` 针对 `State.state` 提供哈希表，用于缓存和去重；`VarMap` 用于递归类型中把类型变量映射到状态。

- `dstate` 与 `DStateSet`（`types.ml:216-248`）  
  确定化自动机节点，与 `State.state` 类似，但 `d_cons`、`d_flow` 使用确定化版本的 `TypeLat.t` 与集合。`DStateSet` 仍基于 `Intmap.Fake`，允许在确定化过程中按照集合键建立映射。

这些结构贯穿后续所有函数：`TypeLat.t` 负责结构性计算，`State.state`/`dstate` 承载自动机节点，`StateSet`/`DStateSet` 实现集合操作。

## 类型 lattice 与辅助结构

- `ty_add p ts`
  1. 过滤掉与极性 `p` 相同的 `TZero` 项。
  2. 根据剩余列表构造 `TZero p` / 单项 / 嵌套 `TAdd (p, …)`，计算并/交正规化。

- 模块 `TypeLat`
  - `LUnexpanded` vs `LExpanded`：`TypeLat.t` 维护“尚未展开的别名”与“已完全展开的基础组件”之间的差异，避免在类型操作中无谓地提前展开别名。
    - `LUnexpanded` 持有单个 `Components.t`，对应非内建、可能需要延迟展开的别名。保留原始结构可在 `expand_both` 时进行 stamp 对齐，以便延迟到真正需要比较时再展开，避免指数级爆炸。
    - `LExpanded` 用列表表示已经被展开成基础组件的集合。对内建类型或已经展开的别名，直接记录其所有构件，方便 join/meet 与遍历。多元素表示同一极性下的并/交分支。
  - `lift`：根据组件 stamp 决定放入 `LExpanded`（内建/已展开）还是 `LUnexpanded`（延迟展开的别名）。
  - `as_list` / `of_list`：在 `LZero`、`LUnexpanded`、`LExpanded` 之间互转；`of_list` 对非内建别名仍保持单元素 `LUnexpanded`，保证延迟策略有效。
  - `pmap`：按极性将组件映射函数 `f` 应用于每个分支；处理函数/对象/基类型的极性翻转。
  - `pfold_comps` / `pfold` / `iter`：折叠工具，用于遍历组件字段或嵌套节点。
  - `join_comps`：将两个组件列表按“同一构件”分组并调用 `Components.join` 合并。
  - `join_ident`：`LZero`，用作 join 运算幺元。
  - `subs_comps`：检查一个列表在极性 `pol` 下是否被另一列表包含（调用 `Components.lte'`）。
  - `list_fields`：罗列结构体/函数的投影字段，用于打印或遍历。
  - `to_typeterm`：把 lattice 结构倒回 `typeterm`（`LZero`→`TZero`，数组→`TAdd`）。
  - `change_locations`：统一替换组件里的 `Location`，用于克隆。

## 状态构造与共享

- `check_polarity s`
  1. 检查 `s.flow` 中所有目标状态极性与 `s.pol` 相反。
  2. 遍历结构约束 `s.cons`，确保每个子节点的极性与所在分支匹配。

- `fresh_id_counter` / `fresh_id`
  - 全局计数器，生成唯一状态编号。

- `mkstate pol cons flow`
  - 创建基础 `state`：设置编号、极性、结构约束 `cons` 与流边集合 `flow`。

- `cons pol t`
  1. 为组件 `t` 创建新状态。
  2. 将组件子结构映射为对每个子状态求 `StateSet.singleton`，再包装成 `TypeLat.lift`。
  3. 初始 `flow` 为空。

- `flow_pair ()`
  1. 建立一对互连的正/负状态（双向 `flow` 边）。
  2. 返回 `(neg_state, pos_state)`，供 join/约束使用。

- `zero p`
  - 构造没有结构约束的极性状态，用于“零”值。

## 组件展开与合并

- `expand_both expa pa ca expb pb cb`
  1. 对左右 `TypeLat` 表达式按 stamp 展开别名，确保两侧在同一层级比较。
  2. 若一方仍是别名则递归展开对应侧。
  3. 返回展开后的 `(ca', cb')`。

- `expand_tybody` / `expand_comp`
  - 递归将 `Typector.tybody` 转化为 `StateSet`/`TypeLat`，用于处理递归类型展开。

- `join p x y`
  1. 先使用 `expand_both` 同步展开。
  2. 调用 `TypeLat.join_comps` + `StateSet.union` 合并结构信息，得到新的 lattice。

- `merge s s'`
  - 假定极性相同，将 `s'` 的结构约束和流边合并进 `s`，并调用 `check_polarity` 验证。

## 类型编译

- `compile_set ctx r pol t`
  1. 对 `typeterm` 递归构建 `StateSet`：
     - `TNamed`：利用 `ty_named` 展开后再递归编译。
     - `TCons`：调用 `compile_cons`。
     - `TRec`：创建临时零状态，递归编译体，最后把结果合并进占位符（检测守护性）。
     - `TZero`/`TAdd`：生成空集或并集。
  2. 上下文 `r` 保存递归变量到状态的映射。

- `compile_cons ctx r pol c`
  - 为组件 `c` 构造新状态，递归编译内部字段，将结果塞入 `TypeLat.lift`。

- `compile_type ctx pol t`
  1. 创建初始零状态。
  2. 编译 `t` 后，把生成的状态集合逐个合并进起始状态。
  3. 返回最终的入口状态。

- `compile_type_pair ctx t`
  1. 特化版本允许 `TWildcard` 返回双极态（`flow_pair`）。
  2. `cons_pair` 负责对组件内的每个子字段递归构建 `(neg, pos)` 对，按极性交换左右。
  3. 其余结构同 `compile_type`。

## 打印与反向解码

- `print_automaton ctx diagram_name ppf map`
  1. 根据 `map` 存入命名节点，递归遍历结构约束，输出 Graphviz 形式。
  2. 流边绘制为虚线，结构边以字段名标注。

- `find_reachable roots`
  - 从给定状态列表出发，依据结构约束递归收集可达节点。

- `garbage_collect root`
  1. 通过 `find_reachable` 找到需要保留的节点。
  2. 将所有流边裁剪到可达集，移除孤立状态。

- `make_table s f`
  - 建立哈希表，将集合中的每个状态映射到 `f` 的结果，供后续查找。

- `decompile_automaton roots`
  1. 通过 `find_reachable`、`make_table` 构建图与流关系。
  2. 求类型流的二分团分解（biclique），为每个团生成类型变量。
  3. 递归遍历状态图：若遇到已访问的节点则创建 `TRec`/命名变量，否则将结构约束与变量求 `ty_add` 后返回。

## 约束收集

- `constrain loc sp sn`
  1. 确认输入极性 (`sp` 正、`sn` 负)。
  2. 以队列形式关闭所有可达约束：同步扩展结构、合并流边。
  3. 对每个正/负结构组合调用 `Components.lte` 检查兼容性，收集 `Error.conflict`。
  4. 返回带源位置信息的错误列表。

## 确定化与最小化

- `mkdstate` / `fresh_dstate`
  - 构造确定化状态结构，初始结构与流均为空。

- `clone f`
  1. 从确定化状态复制成非确定化状态图，保持结构和流关系。
  2. 回调 `f` 接受 `copy_state`，供调用方将 `dstate` 映射回 `state`。

- `determinise old_states`
  1. 使用 `Map<StateSet, dstate>` 将 NFA 集合状态映射为 DFA 节点。
  2. `follow` 递归创建新节点，并把结构通过 `TypeLat.pmap` 继续跟随。
  3. 根据原图的流边生成 DFA 的 `d_flow`。
  4. 返回从单个原始状态到 DFA 状态的映射函数及全部 DFA 节点。

- `minimise dstates`
  1. 对确定化状态按结构与极性建立分区。
  2. 迭代细化分区，直到无法再细分。
  3. 构建 `remap` 函数，将原 DFA 节点映射到代表节点。

- `add_flow`
  - 在 DFA 中添加双向流边（负→正、正→负）辅助后续 entailment。

- `dexpand_tybody` / `dexpand_comp`
  - 与非确定化版本类似，但返回确定化节点，供 entailment 使用。

- `entailed dn dp`
  1. 若目标流已存在直接返回。
  2. 临时加入流边并展开结构，检查负节点的每个分支是否存在一个正节点满足 `lte'`。
  3. 若成立则保留流边，否则恢复原状并返回 `false`。

- `subsumed map`
  1. `map` 回调用于对所有候选子图执行 `subsume` 判定。
  2. `subsume` 调用 `expand_both` 比较结构；`subsume_all` 负责遍历集合。
  3. 同步校验数据流：若负节点已知有流到某集合，则要求对应正节点间存在 `entailed` 关系。
  4. 返回整体是否满足包含关系。

- `find_reachable_dstates`
  - 类似 `find_reachable`，但针对确定化节点并沿结构约束递归。

- `optimise_flow`
  1. 收集所有负极流边，按逆后序处理。
  2. 清空流关系，逐条尝试加入；若 `entailed` 返回 `true` 表示冗余，跳过。
  3. 最终仅保留必要的流边。
