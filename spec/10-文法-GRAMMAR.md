# Cosat 文法汇总（EBNF）

**版本**：Cosat v0.1 ｜ **日期**：2026-08-25
**状态**：🔒 设计定案（标注 ⏳ / ❓ 处除外）｜ **相关**：各章规范 ｜ 动机见 [design/](../design/README.md)

本档为全仓形式文法（EBNF）的唯一事实源，按章组织，各章以链接引用；语义约束（点位校验、穷尽性、禁写清单）以各章规范为准。

---

## §10.1 词法（[01](./01-词法-LEXICAL.md)）

```ebnf
comment-line  = "#" , { character - newline } ;
comment-block = "#$" , { character } , "$#" ;
number        = digit-seq , [ "." , digit-seq ] ;   (* 含 `-3.14` 形态（一元负号）；`_` 分隔符仅限两数字之间 *)
digit-seq     = digit , { ( digit | "_" , digit ) } ;
escape        = "\" , ( '"' | "\" | "n" | "t" | "r" ) ;   (* 转义集封闭：未知转义编译错 *)
string        = '"' , { ( character - ( newline | '"' | "\" ) ) | escape } , '"' ;   (* 单行；原始换行编译错 *)
boolean       = "true" | "false" ;
identifier    = ( letter | "_" | "$" ) , { letter | digit | "_" | "$" } ;
keyword       = "def" | "when" | "match" | "each" | "while" ;
operator      = "->" | "=>" | "==" | "<" | ">" | "*" | "/" | "+" | "-" | "!" | "@" | ":" | ";" | "," | "." ;
delimiter     = "{" | "}" | "(" | ")" ;
placeholder   = "_" ;
```

**扫描级规则**（非 CFG，实现为谓词）：

| 规则 | 内容 |
| --- | --- |
| 前缀空格 🔒 | `\` 后必须空格（`@\` 复合前缀例外，`\` 后仍须空格）；紧贴形式编译错；关键词构造 `when` `match` `each` `while` 以 token 边界自分隔，无此规则 |
| 体边界 🔒 | 一切位置统一：单行 lambda 体自 `=>` 后扫描到所在文法位的第一个未配对封闭符——语句位 / 站点位 `->` 或 `;`；分支体位 `;` / `}`；循环体位 `}`；括号内 `)`；配对块 / 括号内不截断。单行体内不能裸写管道（块体 / 括号内自由） |
| 守卫自定界 🔒 | 分支左侧扫描到 `:` 止 |
| 被匹配值 / 源扫描 🔒 | 扫描到 `{` 止，跳过配对块与括号；内禁 `->` `;` `,` `:` |
| 数字分隔符 🔒 | `_` 仅允许出现于两个数字之间；`1000_`、`1__000` 编译错；`_1000` 为标识符（首字符分流，语义见 [09-数据类型](./09-数据类型-TYPES.md) §9.2） |
| 转义集封闭 🔒 | 字符串内反斜杠只认 escape 产生式所列五项，未知转义编译错（[09-数据类型](./09-数据类型-TYPES.md) §9.3） |
| 字符串单行 🔒 | 字符串字面量不得跨行，原始换行编译错；多行形态 ⏳ 归 [11-路线图](./11-路线图-ROADMAP.md) |

## §10.2 程序结构（[08](./08-语句-STATEMENTS.md)）

```ebnf
program      = { statement } ;
statement    = chain , ";"
             | match-expr , ";"          (* match / when：语句位可省 _ 穷尽 *)
             | when-expr , ";"
             | each-expr , ";"
             | while-expr , ";"
             | operand , ";" ;
block        = "{" , { statement } , [ tail-operand ] , "}" ;
tail-operand = operand ;                             (* 无分号 = 块值出口；带分号 → 块值为空值 *)
```

定义不是独立语句：lambda 链首 + `-> def 名字` 段即定义（§10.4）。

## §10.3 表达式与运算符（[03](./03-运算符-OPERATORS.md)）

优先级：`* /` ＞ `+ -` ＞ `< >` ＞ `==` ＞ [`?:` 三元 ⏳（待定；若定案则最低）]；单目与前缀 `@` 高于一切二元。结合性未规定 ⏳，暂按左结合实现。存储不是运算符。

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
             | lambda | match-expr | when-expr | each-expr | while-expr ;  (* 后四者限值位开头 *)
```

## §10.4 链（[04](./04-链模型-CHAIN.md)）

```ebnf
chain        = [ value ] , { segment } ;
value        = operand | param-group ;
param-group  = operand , { "," , operand } ;
segment      = "->" , station
             | "->" , recv-group , "->" , station ;
recv-group   = recv-item , { "," , recv-item } ;
recv-item    = at-expr | def-binding | "_" | operand ;
station      = at-expr
             | "(" , "@@" , operand , ")"
             | def-binding
             | "match" , "_" , branch-block
             | "each" , "_" , block ;
def-binding  = "def" , identifier ;                        (* tee：绑定并透传，恰 1 位 *)
```

**点位计数（语义）**：`@fn` = fn 形参数量（不跨层）；裸 `_` 与 `def 名字` = 1；嵌套 `match _` / `each _` = 1；裸引用 / 字面量 / 普通表达式（含括号包裹）/ when / while 嵌套 = 0——带洞运算式（`_ + 1` 类）不作成员，匿名消费仅认裸 `_`，构造内部的引用型见 [06](./06-匹配-MATCH.md)。成员从左到右逐位消费，0 位成员跳过。每跳校验：点位总数 = 上游参数数量。

**def 语义**：绑定在所在作用域、可见性从绑定点起、无变异（同名 = 遮蔽式新绑定）、链式合法（`1 -> def x -> def y;`）。见 [design/09](../design/09-链上定义.md)。

## §10.5 函数定义与调用（[05](./05-函数-FUNCTIONS.md)）

```ebnf
lambda       = "\" , params , "=>" , body ;
params       = { param } ;                                (* 可空：\ => 42 *)
param        = identifier | "_" ;
body         = operand | block ;
```

定义形态：`\ 形参 => 体 -> def 名字`（单行体到 `->` 止）。多层调用右结合：`@@fn` = `@(@fn)` = `fn(data)()`。

## §10.6 match 与 when（[06](./06-匹配-MATCH.md)）

```ebnf
match-expr   = "match" , scrutinee , branch-block ;   (* 值分发：必有被匹配值 *)
when-expr    = "when" , branch-block ;                (* 条件分发：无被匹配值 *)
scrutinee    = operand-banned ;
branch-block = "{" , branch , { branch } , "}" ;
branch       = branch-left , ":" , receiver ;
branch-left  = "_" | guard-operand | literal ;
guard-operand = operand-with-"_" ;                    (* match 守卫须含 _ *)
receiver     = operand | block ;
```

构造约束：match 守卫须含 `_`（字面量分支除外）、单独 `_` 为 default、被匹配值求值一次；when 守卫为任意布尔表达式且禁 `_`、体须 0 位；分支 ≥ 1；无 fallthrough；值位使用须含单独 `_` default 且有值出口。

## §10.7 each 与 while（[07](./07-循环-LOOP.md)）

```ebnf
each-expr    = "each" , source , block            (* 遍历型 *)
             | "each" , source , guard-block      (* 遍历带守卫型 *)
             | "each" , "_" , block ;             (* 站点源，仅链站点位 *)
while-expr   = "while" , guard-block ;            (* 条件循环，仅语句位 / 值位 *)
source       = operand-banned ;
guard-block  = "{" , guard-branch , { guard-branch } , "}" ;
guard-branch = guard , ":" , receiver ;
receiver     = operand | block ;                  (* each 供 1 值；while 供 0 值 *)
guard        = "_" | operand-with-"_" ;
```

while 守卫禁 `_`（单独 `_` 守卫 v1 编译错，随 break 解禁）、每轮重评守卫、不可入链；遍历带守卫型守卫 `_` = 当前元素；收集开关 = 体尾分号（Block.tail）；无内部变异状态机形态（递归表达或待远期显式容器）。

## §10.8 共享终结符约定

| 记号 | 含义 |
| --- | --- |
| `operand` | 普通表达式（§10.3 产生式可达的式子） |
| `operand-banned` | 同上，但扫描区域禁 `->` `;` `,` `:`（见 §10.1 扫描级规则） |
| `operand-with-"_"` | 至少含一处 `_` 的表达式（match 守卫要求） |
| `literal` | number / string / boolean |
