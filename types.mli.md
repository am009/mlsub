# types.mli 接口说明

`types.mli` 暴露类型自动机（Type Automata）相关操作，供类型推断、签名打印等模块调用。以下按声明顺序解释接口含义。

## 基本状态类型
- `type state`  
  非确定性自动机中的节点类型，隐藏内部结构，仅能通过接口函数操作。

## 构造与编译
- `val cons : polarity -> state Typector.Components.t -> state`  
  根据极性与 `Components` 结构创建新节点，建立结构约束。
- `val compile_type : context -> polarity -> typeterm -> state`  
  将类型语法树编译为自动机入口节点，使用给定上下文解析具名类型。
- `val compile_type_pair : context -> typeterm -> state * state`  
  编译一个类型为一对（负极、正极）状态，常用于函数参数与结果的双引导。

## 调试辅助
- `val print_automaton : context -> string -> Format.formatter -> ((string -> state -> unit) -> unit) -> unit`  
  以 Graphviz 风格打印自动机，回调提供额外命名节点。
- `val garbage_collect : state -> unit`  
  清理从入口不可达的节点，减少内存占用。
- `val decompile_automaton : state list -> typeterm list`  
  将自动机状态集合还原为语法层类型表示（用于输出推断结果）。

## 预置工具
- `val flow_pair : unit -> state * state`  
  创建互联的（负、正）状态对，方便建立入/出流约束。
- `val zero : polarity -> state`  
  返回指定极性的空节点。

## 约束与错误
- `val constrain : Location.t -> state -> state -> Error.t list`  
  施加子类型约束（正 ≤ 负），返回冲突列表并附带源位置。

## 确定化状态类型
- `type dstate`  
  确定化自动机中的节点类型，同样对外隐藏内部结构。

## 自动机转换与优化
- `val clone : ((Location.set -> dstate -> state) -> 'a) -> 'a`  
  将确定化自动机复制成非确定性版本，回调中可使用 `dstate -> state` 的转换函数。
- `val determinise : state list -> (state -> dstate) * dstate list`  
  将一个或多个 `state` 组成的 NFA 转换为 DFA，返回映射函数及所有新节点。
- `val minimise : dstate list -> dstate -> dstate`  
  对 DFA 进行最小化，给出代表节点映射函数。
- `val entailed : dstate -> dstate -> bool`  
  判断一条负→正流是否由现有结构必然成立，用于消除冗余流边。
- `val subsumed : ((polarity -> state -> dstate -> bool) -> bool) -> bool`  
  提供回调，用于检查一组包含关系是否全部满足（结合 `entailed` 和结构比较）。
- `val find_reachable_dstates : dstate list -> dstate list`  
  从入口集合出发收集结构可达节点。
- `val optimise_flow : dstate list -> unit`  
  对 DFA 的流边做冗余消除，仅保留必要的双向链接。
