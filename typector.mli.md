# typector.mli 接口说明

`typector.mli` 暴露类型构造与上下文管理的接口，供类型推断、打印等模块调用。本文件按出现顺序解释每个声明的语义。

## 基础类型与别名
- `type 'a printer = Format.formatter -> 'a -> unit`  
  格式化输出函数类型别名，方便在接口中复用。
- `module SMap = Symbol.Map`  
  将符号映射实例化成局部别名，便于在接口里引用。

## 冲突报告
- `type conflict = Location.set * Location.set * Error.conflict_reason`  
  表示两个定位集合之间的类型冲突及其原因，用于子类型比较时的错误诊断。

## `Components` 子模块
提供以函数、对象、具名基类型三种形态为核心的类型组件操作。
- `type +'a t`  
  多态组件类型，参数 `'a` 通常代表子节点的构造方式。
- `val cmp_component : 'a t -> 'b t -> bool`  
  判断两个组件是否同构（参数数量、结构是否相符）。
- `val join : polarity -> (polarity -> 'a -> 'a -> 'a) -> 'a t -> 'a t -> 'a t`  
  在指定极性下合并组件。回调函数负责合并子节点。
- `val lte : ... -> conflict list` / `val lte' : ... -> bool`  
  分别返回详细冲突信息或布尔结果，用于子类型/上界检查。
- `val pmap` / `val pfold`  
  极性感知的映射与折叠操作，可保持或反转子节点的极性。
- `val list_fields`  
  将组件展开为字段/参数列表配合调试或打印使用。
- `val change_locations`  
  批量调整组件携带的源位置信息，常用于克隆。

## 类型语法结构
- `type tyvar = Symbol.t`  
  类型变量名称即符号。
- `type typaram`  
  表示类型构造器的形式参数，可指定变型约束或匿名占位。
- `type +'a tyarg` 与 `type +'a tyargterm`  
  描述类型实参：正向、负向、双向或缺省；`tyargterm` 额外区分是否显式标注变型。
- `type typeterm`  
  类型语法树：零类型（顶/底）、具名类型、构造类型、并/交、递归与通配符。

## 类型体与上下文
- `type +'a tybody = BParam of 'a | BCons of 'a tybody Components.t`  
  在别名展开时使用的中间结构，保留参数与子组件。
- `type stamp = private int`  
  类型定义的内部编号，使用 private 限制外部构造。
- `val builtin_stamp : stamp`  
  内建或完全展开类型的 stamp 基准。
- `type context` / `val empty_context`  
  类型定义上下文，维护符号与 stamp 的映射。

## 上下文扩展接口
- `val add_type_alias`  
  向上下文注册类型别名：提供名称、参数列表、别名体 `typeterm`。
- `val add_opaque_type`  
  注册不透明类型，外部只能使用其名称和变型信息。

## 别名信息读取
- `val get_stamp`  
  取得组件的 stamp，别名/不透明类型区分与内建类型。
- `val expand_alias`  
  在别名场景下展开组件，返回以 `tybody` 表示的结构（强制具守护性）。
- `val find_by_name`  
  根据符号查找定义，得到 `(stamp, variance list)` 或 `None`。
- `val name_of_stamp`  
  将 stamp 反查回原始符号。
- `val print_typeterm`  
  生成打印 `typeterm` 的格式化函数，需提供上下文以解析具名类型。

## 构造具体组件
这些函数将语法层的描述转换为 `Components.t`：
- `val ty_fun`  
  构造函数类型组件，输入位置参数、关键字参数和返回类型的生成器。
- `val ty_obj_cases` / `val ty_obj_tag` / `val ty_obj` / `val ty_obj_l`  
  构造对象/记录类型组件，支持标签化分支或匿名记录。
- `val ty_named` / `val ty_named'` / `val ty_base`  
  解析具名类型调用：前两者根据上下文查找并映射参数，`ty_base` 直接通过 stamp 构造组件。
