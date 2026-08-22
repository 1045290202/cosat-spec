# Cosat 文法汇总（EBNF）

**版本**：Cosat v0.1 ｜ **日期**：2026-08-21
**状态**：🔒 设计定案（标注 ⏳ / ❓ 处除外）｜ **相关**：各章规范 ｜ 动机见 [design/](../design/README.md)

本文档为全量文法速览，按章组织；语义约束（点位校验、穷尽性、禁写清单）以各章规范为准。

---

## §12.1 词法（[01](./01-词法-LEXICAL.md)）

```ebnf
comment-line  = "#" , { character - newline } ;
comment-block = "#$" , { character } , "$#" ;
number        = digit , { digit } , [ "." , digit , { digit } ] ;
string        = '"' , { character } , '"' ;                          (* 无转义 ⏳ *)
boolean       = "true" | "false" ;
identifier    = ( letter | "_" | "$" ) , { letter | digit | "_" | "$" } ;
keyword       = "def" ;
operator      = ">>" | "->" | "=>" | "==" | "<" | ">" | "*" | "/" | "+" | "-" | "!" | "@" | "?" | "~" | ":" | ";" | "," | "." ;
delimiter     = "{" | "}" | "(" | ")" ;
placeholder   = "_" ;
```

**扫描级规则**（非 CFG，实现为谓词）：

| 规则 | 内容 |
| --- | --- |
| 前缀空格 🔒 | `\` `?` `~` 后必须空格（`@\` 复合前缀例外，`\` 后仍须空格）；紧贴形式编译错 |
| 贪婪吞并 ⏳ | 运算符与 `;` 相邻合并报错（`->;`），运算符后须空格；改进方向：最长匹配 + 回退 |
| 体边界 🔒 | 单行 lambda 体自 `=>` 后扫描到第一个未配对终止符：语句位 `>>`/`;`；站点位 `->`/`>>`/`;`；分支体位 `;`/`}`；循环体位 `}`；括号内 `)`；配对块/括号内不截断 |
| 守卫自定界 🔒 | 分支左侧扫描到 `:` 止 |
| 被匹配值/源扫描 🔒 | 扫描到 `{` 止，跳过配对块与括号；内禁 `->` `>>` `;` `,` `:` |

## §12.2 程序结构（[08](./08-语句-STATEMENTS.md)）

```ebnf
program      = { statement } ;
statement    = chain , ";"
             | definition , ";"
             | match-expr , ";"
             | loop-expr , ";"
             | operand , ";" ;
block        = "{" , { statement } , [ tail-operand ] , "}" ;
tail-operand = operand ;
```

## §12.3 表达式与运算符（[03](./03-运算符-OPERATORS.md)）

优先级：`< >`(5) ＞ `* /`(4) ＞ `+ -`(3) ＞ `==`(2) ＞ [`?:` 三元(1.5) ⏳] ＞ `>>`(1)；单目与前缀 `@` 高于一切二元。结合性未规定 ⏳。

```ebnf
expr         = equality ;
equality     = comparison , [ "==" , comparison ] ;
comparison   = additive , [ ( "<" | ">" ) , additive ] ;
additive     = term , { ( "+" | "-" ) , term } ;
term         = unary , { ( "*" | "/" ) , unary } ;
unary        = ( "+" | "-" | "!" ) , unary | at-expr | primary ;
at-expr      = "@" , callee ;
callee       = dotted-name | at-expr | lambda-inline ;
dotted-name  = identifier , { "." , identifier } ;
lambda-inline= "\" , params , "=>" , body ;
primary      = number | string | boolean | identifier | "_" | "(" , chain , ")"
             | lambda | match-expr | loop-expr ;          (* 后三者限值位开头 *)
```

## §12.4 链（[04](./04-链模型-CHAIN.md)）

```ebnf
chain        = [ value ] , { segment } ;
value        = operand | param-group ;
param-group  = operand , { "," , operand } ;
segment      = "->" , station
             | "->" , recv-group , "->" , station ;
recv-group   = recv-item , { "," , recv-item } ;
recv-item    = at-expr | "_" | operand ;
station      = at-expr
             | "(" , "@@" , operand , ")"
             | "?" , "_" , branch-block
             | "~" , "_" , block ;
```

**点位计数（语义）**：`@fn` = fn 形参数量（不跨层）；`_` = 1；裸引用 / 字面量 / cond 型嵌套 = 0；嵌套 `? _` / `~ _` = 1。每跳校验：点位总数 = 上游参数数量。

## §12.5 函数定义与调用（[05](./05-函数-FUNCTIONS.md)）

```ebnf
definition   = lambda , ">>" , "def" , identifier ;
lambda       = "\" , params , "=>" , body ;
params       = { param } ;
param        = identifier | "_" ;
body         = operand | block ;
```

多层调用右结合：`@@fn` = `@(@fn)` = `fn(data)()`。

## §12.6 match（[06](./06-匹配-MATCH.md)）

```ebnf
match-expr   = "?" , [ scrutinee ] , branch-block ;
scrutinee    = operand-banned ;
branch-block = "{" , branch , { branch } , "}" ;
branch       = branch-left , ":" , receiver ;
branch-left  = "_" | guard-operand | literal ;
receiver     = operand | block ;
```

分型约束：cond 型（无 scrutinee）守卫禁 `_`、体须 0 位；switch 型守卫须含 `_`（字面量除外）、单独 `_` 为 default；型内禁混；分支 ≥ 1；无 fallthrough；值位须含单独 `_` default 且有值出口。

## §12.7 循环（[07](./07-循环-LOOP.md)）

```ebnf
loop-expr    = "~" , loop-form ;
loop-form    = source , block            (* 遍历型 *)
             | source , guard-block      (* 遍历带守卫型 *)
             | guard-block               (* 条件型 *)
             | "_" , block ;             (* 站点源，仅链站点位 *)
source       = operand-banned ;
guard-block  = "{" , guard-branch , { guard-branch } , "}" ;
guard-branch = guard , ":" , receiver ;
guard        = "_" | operand-with-"_" ;
```

条件型守卫禁 `_`（单独 `_` 守卫 v1 编译错，随 break 解禁）；遍历带守卫型守卫 `_` = 当前元素；收集开关 = 体尾分号（Block.tail）。

## §12.8 共享终结符约定

| 记号 | 含义 |
| --- | --- |
| `operand` | 普通表达式（§12.3 产生式可达的式子） |
| `operand-banned` | 同上，但扫描区域禁 `-> >> ; , :`（见 §12.1 扫描级规则） |
| `operand-with-"_"` | 至少含一处 `_` 的表达式（switch 型守卫要求） |
| `literal` | number / string / boolean |
