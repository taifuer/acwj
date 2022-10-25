# Part 21：更多运算符

在我们编译器编写旅程的这一部分，我决定挑选一些唾手可得的果实，并实现许多仍然缺失的表达式运算符。其中包括：

- `++`和`--`，前自增/自减和后自增/自减
- 一元 `-`、`~`、和`!`
- 二元`^`，`&`，`|`，`<<`和`>>`

我还实现了隐式的“非零运算符”，它将表达式右值视为选择和循环语句的布尔值，例如：

```
  for (str= "Hello"; *str; str++) ...
```

而不是写成：

```c
  for (str= "Hello"; *str != 0; str++) ...
```

## 标记和扫描

和往常一样，我们从语言中的任何新标记开始。这次有几个：

| 扫描的输入 | 标记     |
| ---------- | -------- |
| `||`       | T_LOGOR  |
| `&&`       | T_LOGAND |
| `|`        | T_OR     |
| `^`        | T_XOR    |
| `<<`       | T_LSHIFT |
| `>>`       | T_RSHIFT |
| `++`       | T_INC    |
| `--`       | T_DEC    |
| `~`        | T_INVERT |
| `!`        | T_LOGNOT |

其中一些运算符由新的单个字符组成，因此扫描这些字符很容易。对于其他运算符，我们需要区分单个字符和成对的不同字符。例如`<`、`<<`和`<=`。我们已经看到了如何在`scan.c`中对这些进行扫描。所以我不会在这里给出新代码。浏览扫描`scan.c`查看添加内容。

## 将二元运算符添加到解析中

现在我们需要解析这些运算符。其中一些运算符是二元运算符：`||`、`&&`、`|`、`^`、`<<`和`> >`。我们已经为二元运算符准备了一个优先级框架。我们可以简单地将新的运算符添加到框架中。

当我这样做时，我意识到根据这个 [C 运算符优先级表](https://en.cppreference.com/w/c/language/operator_precedence)，我有几个现有的运算符具有错误的优先级。我们还需要使 AST 节点运算与二元运算符标记集保持一致。因此，下面是标记、AST 节点类型和运算符优先级表的定义，来自`defs.h`和`expr.c`：

```c
// Token types
enum {
  T_EOF,
  // Binary operators
  T_ASSIGN, T_LOGOR, T_LOGAND, 
  T_OR, T_XOR, T_AMPER, 
  T_EQ, T_NE,
  T_LT, T_GT, T_LE, T_GE,
  T_LSHIFT, T_RSHIFT,
  T_PLUS, T_MINUS, T_STAR, T_SLASH,

  // Other operators
  T_INC, T_DEC, T_INVERT, T_LOGNOT,
  ...
};

// AST node types. The first few line up
// with the related tokens
enum {
  A_ASSIGN= 1, A_LOGOR, A_LOGAND, A_OR, A_XOR, A_AND,
  A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE, A_LSHIFT, A_RSHIFT,
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE,
  ...
  A_PREINC, A_PREDEC, A_POSTINC, A_POSTDEC,
  A_NEGATE, A_INVERT, A_LOGNOT,
  ...
};

// Operator precedence for each binary token. Must
// match up with the order of tokens in defs.h
static int OpPrec[] = {
  0, 10, 20, 30,                // T_EOF, T_ASSIGN, T_LOGOR, T_LOGAND
  40, 50, 60,                   // T_OR, T_XOR, T_AMPER 
  70, 70,                       // T_EQ, T_NE
  80, 80, 80, 80,               // T_LT, T_GT, T_LE, T_GE
  90, 90,                       // T_LSHIFT, T_RSHIFT
  100, 100,                     // T_PLUS, T_MINUS
  110, 110                      // T_STAR, T_SLASH
};
```

## 新的一元运算符

现在我们开始解析新的一元运算符`++`、`--`、`~`和`!`。所有这些都是前缀运算符（即在表达式之前），但是`++`和`--`运算符也可以是后缀运算符。因此，我们需要解析三个（四个？）前缀和两个后缀运算符，并为它们执行五种不同的语义操作。

为了准备这些新运算符的加入，我回去查阅了 [C 的 BNF 语法](https://www.lysator.liu.se/c/ANSI-C-grammar-y.html)。由于这些新运算符不能被应用到现有的二元运算符框架中，我们需要在递归下降解析器中使用新函数来实现它们。下面是上面语法的相关部分，重写后使用我们的标记名：


```c
primary_expression
        : T_IDENT
        | T_INTLIT
        | T_STRLIT
        | '(' expression ')'
        ;

postfix_expression
        : primary_expression
        | postfix_expression '[' expression ']'
        | postfix_expression '(' expression ')'
        | postfix_expression '++'
        | postfix_expression '--'
        ;

prefix_expression
        : postfix_expression
        | '++' prefix_expression
        | '--' prefix_expression
        | prefix_operator prefix_expression
        ;

prefix_operator
        : '&'
        | '*'
        | '-'
        | '~'
        | '!'
        ;

multiplicative_expression
        : prefix_expression
        | multiplicative_expression '*' prefix_expression
        | multiplicative_expression '/' prefix_expression
        | multiplicative_expression '%' prefix_expression
        ;

        etc.
```

我们在`expr.c`中实现了`binexpr()`中的二元运算符，但这调用了`prefix()`，就像上面的 BNF 语法中的`multiplicative_expression`引用的是`prefix_expression`一样。我们已经有了一个名为`primary()`的函数。现在我们需要一个函数`postfix()`来处理后缀表达式。

## 前缀运算符

我们已经解析了`prefix()`中的两个标记：T_AMPER 和 T_STAR。通过向 `switch (Token.token)` 语句中添加更多 case 语句，我们可以在这里添加新的标记（T_MINUS，T_INVERT，T_LOGNOT，T_INC 和 T_DEC）。

我不会在这里包含代码，因为所有的案例都有一个相似的结构：

- 使用 `scan(&Token)` 跳过标记
- 用 `prefix()` 解析下一个表达式
- 做一些语义检查
- 扩展由 `prefix()` 返回的 AST 树

然而，其中一些案例之间的差异很重要。对于`&`(T_AMPER)标记的解析，表达式需要被视为左值：如果我们做`&x`，我们想要变量 `x` 的地址，而不是 `x` 的值的地址。其他情况确实需要将`prefix()`返回的 AST 树强制为右值：

- `-` (T_MINUS)
- `~` (T_INVERT)
- `!` (T_LOGNOT)

而且，对于前自增和前自减运算符，我们实际上要求表达式是左值：我们可以做`++x`，但不能做`++3`。目前，我已经编写了需要一个简单标识符的代码，但我知道稍后我们将需要解析和处理`++b[2]`和`++ *ptr`。

此外，从设计的角度来看，我们可以选择更改 `prefix()` 返回的 AST 树（不添加新的 AST 节点），或者向树中添加一个或多个新的 AST 节点：

- T_AMPER 修改现有的 AST 树，因此根是 A_ADDR
- T_STAR 将一个 A_DEREF 节点添加到树的根
- T_STAR 在可能将树扩展为 `int` 值之后，将 A_NEGATE 节点添加到树的根。为什么？因为树可能是无符号的 `char` 类型，你不能对无符号值进行反求。
- T_INVERT 将 A_INVERT 节点添加到树的根
- T_LOGNOT 将 A_LOGNOT 节点添加到树的根
- T_INC 将一个 A_PREINC 节点添加到树的根
- T_DEC 将一个 A_PREDEC 节点添加到树的根

## 解析后缀运算符

如果你看了我上面超链接的 BNF 语法，要解析后缀表达式，我们需要引用主表达式的解析。要实现这一点，我们需要首先获取主表达式的标记，然后确定是否有任何尾随后缀标记。

尽管语法显示"postfix"调用"primary"，但我已经通过扫描 `primary()` 中的标记来实现它，然后决定调用 `postfix()` 来解析后缀标记。

> 事实证明，这是一个错误 -- Warren, writing from the future.

上面的 BNF 语法似乎允许像`x++ ++`这样的表达式，因为它有：

```
postfix_expression:
        postfix_expression '++'
        ;
```

但我不允许表达式后面有多个后缀运算符。让我们看看新代码：

`primary()`处理识别主表达式：整型字面量、字符串字面量和标识符。它还能识别插入括号的表达式。只有标识符后面可以跟后缀运算符。

```c
static struct ASTnode *primary(void) {
  ...
  switch (Token.token) {
    case T_INTLIT: ...
    case T_STRLIT: ...
    case T_LPAREN: ...
    case T_IDENT:
      return (postfix());
    ...
}
```

我已经将函数调用和数组引用的解析移到`postfix()`，这是我们解析后缀`++`和`--`运算符的地方：

```c
// Parse a postfix expression and return
// an AST node representing it. The
// identifier is already in Text.
static struct ASTnode *postfix(void) {
  struct ASTnode *n;
  int id;

  // Scan in the next token to see if we have a postfix expression
  scan(&Token);

  // Function call
  if (Token.token == T_LPAREN)
    return (funccall());

  // An array reference
  if (Token.token == T_LBRACKET)
    return (array_access());


  // A variable. Check that the variable exists.
  id = findglob(Text);
  if (id == -1 || Gsym[id].stype != S_VARIABLE)
    fatals("Unknown variable", Text);

  switch (Token.token) {
      // Post-increment: skip over the token
    case T_INC:
      scan(&Token);
      n = mkastleaf(A_POSTINC, Gsym[id].type, id);
      break;

      // Post-decrement: skip over the token
    case T_DEC:
      scan(&Token);
      n = mkastleaf(A_POSTDEC, Gsym[id].type, id);
      break;

      // Just a variable reference
    default:
      n = mkastleaf(A_IDENT, Gsym[id].type, id);
  }
  return (n);
}
```

另一个设计决策。对于 `++`，我们可以用 A_POSTINC 父节点创建一个 A_IDENT AST 节点，但是鉴于我们在 Text 中有标识符的名称，我们可以构建一个单独的 AST 节点，其中既包含节点类型，也包含对符号表中标识符槽号的引用。

## 将整数表达式转换为布尔值

在我们结束解析并转移到代码生成方面之前，我应该提到我所做的更改，即允许将整数表达式视为布尔表达式。

```c
  x= a + b;
  if (x) { printf("x is not zero\n"); }
```

BNF 语法没有提供任何显式的语法规则来限制表达式为布尔型，例如:

```c
selection_statement
        : IF '(' expression ')' statement
```

因此，我们必须从语义上进行处理。在`stmt.c`中，我解析了 IF，WHILE 和 FOR 循环，我添加了以下代码：

```c
  // Parse the following expression
  // Force a non-comparison expression to be boolean
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    condAST = mkastunary(A_TOBOOL, condAST->type, condAST, 0);
```

我引入了一个新的 AST 节点类型 A_TOBOOL。这将生成接受任何整数值的代码。如果该值为 0，则结果为 0，否则结果为 1。

## 为新运算符生成代码

现在我们将注意力转向为新运算符生成代码。实际上，新的 AST 节点类型为：A_LOGOR、A_LOGAND、A_OR、A_XOR、A_AND、A_LSHIFT、A_RSHIFT，A_PREINC、A_PREDEC、A_POSTINC、A_POSTDEC、A_NEGATE、A_INVERT、A_LOGNOT 和 A_TOBOOL。

所有这些都是对 `cg.c` 中平台特定代码生成器中匹配函数的简单调用。因此，`gen.c`中`genAST()`中的新代码很简单：

```c
    case A_AND:
      return (cgand(leftreg, rightreg));
    case A_OR:
      return (cgor(leftreg, rightreg));
    case A_XOR:
      return (cgxor(leftreg, rightreg));
    case A_LSHIFT:
      return (cgshl(leftreg, rightreg));
    case A_RSHIFT:
      return (cgshr(leftreg, rightreg));
    case A_POSTINC:
      // Load the variable's value into a register,
      // then increment it
      return (cgloadglob(n->v.id, n->op));
    case A_POSTDEC:
      // Load the variable's value into a register,
      // then decrement it
      return (cgloadglob(n->v.id, n->op));
    case A_PREINC:
      // Load and increment the variable's value into a register
      return (cgloadglob(n->left->v.id, n->op));
    case A_PREDEC:
      // Load and decrement the variable's value into a register
      return (cgloadglob(n->left->v.id, n->op));
    case A_NEGATE:
      return (cgnegate(leftreg));
    case A_INVERT:
      return (cginvert(leftreg));
    case A_LOGNOT:
      return (cglognot(leftreg));
    case A_TOBOOL:
      // If the parent AST node is an A_IF or A_WHILE, generate
      // a compare followed by a jump. Otherwise, set the register
      // to 0 or 1 based on it's zeroeness or non-zeroeness
      return (cgboolean(leftreg, parentASTop, label));
```

## x86-64 特定代码生成函数

这意味着我们现在可以查看后端函数来生成真正的 x86-64 汇编代码。对于大多数按位操作，x86-64 平台都有汇编指令来执行这些操作：

```c
int cgand(int r1, int r2) {
  fprintf(Outfile, "\tandq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1); return (r2);
}

int cgor(int r1, int r2) {
  fprintf(Outfile, "\torq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1); return (r2);
}

int cgxor(int r1, int r2) {
  fprintf(Outfile, "\txorq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1); return (r2);
}

// Negate a register's value
int cgnegate(int r) {
  fprintf(Outfile, "\tnegq\t%s\n", reglist[r]); return (r);
}

// Invert a register's value
int cginvert(int r) {
  fprintf(Outfile, "\tnotq\t%s\n", reglist[r]); return (r);
}
```

对于移位操作，据我所知，移位量必须首先加载到`%cl`寄存器中。

```c
int cgshl(int r1, int r2) {
  fprintf(Outfile, "\tmovb\t%s, %%cl\n", breglist[r2]);
  fprintf(Outfile, "\tshlq\t%%cl, %s\n", reglist[r1]);
  free_register(r2); return (r1);
}

int cgshr(int r1, int r2) {
  fprintf(Outfile, "\tmovb\t%s, %%cl\n", breglist[r2]);
  fprintf(Outfile, "\tshrq\t%%cl, %s\n", reglist[r1]);
  free_register(r2); return (r1);
}
```

处理布尔表达式（结果必须是 0 或 1）的操作稍微复杂一些。

```c
// Logically negate a register's value
int cglognot(int r) {
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r], reglist[r]);
  fprintf(Outfile, "\tsete\t%s\n", breglist[r]);
  fprintf(Outfile, "\tmovzbq\t%s, %s\n", breglist[r], reglist[r]);
  return (r);
}
```

`test ` 指令实际上与寄存器本身进行 AND，以设置零和负标志。然后，如果它等于零（`sete`），我们将寄存器设置为 1。然后我们将这个 8 位结果移到 64 位寄存器中。

下面是将整数转换为布尔值的代码：

```c
// Convert an integer value to a boolean value. Jump if
// it's an IF or WHILE operation
int cgboolean(int r, int op, int label) {
  fprintf(Outfile, "\ttest\t%s, %s\n", reglist[r], reglist[r]);
  if (op == A_IF || op == A_WHILE)
    fprintf(Outfile, "\tje\tL%d\n", label);
  else {
    fprintf(Outfile, "\tsetnz\t%s\n", breglist[r]);
    fprintf(Outfile, "\tmovzbq\t%s, %s\n", breglist[r], reglist[r]);
  }
  return (r);
}
```

同样，我们做一个`test`来获得寄存器的零或非零。如果是针对选择语句或循环语句执行此操作，那么如果结果为假，则`je`将跳转。否则，使用`setnz`将寄存器设置为 1，如果它最初是非零的。

## 自增和自减运算符

我把`++`和`--`操作留到最后。这里的微妙之处在于，我们既要将值从内存位置取出到寄存器中，又要分别对其进行加或减。我们必须在加载寄存器之前或之后选择这样做。

因为我们已经有了`cgloglob()`函数来加载全局变量的值，所以让我们根据需要修改它来修改变量。代码很难看，但它确实有效。

```c
// Load a value from a variable into a register.
// Return the number of the register. If the
// operation is pre- or post-increment/decrement,
// also perform this action.
int cgloadglob(int id, int op) {
  // Get a new register
  int r = alloc_register();

  // Print out the code to initialise it
  switch (Gsym[id].type) {
    case P_CHAR:
      if (op == A_PREINC)
        fprintf(Outfile, "\tincb\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_PREDEC)
        fprintf(Outfile, "\tdecb\t%s(\%%rip)\n", Gsym[id].name);
      fprintf(Outfile, "\tmovzbq\t%s(%%rip), %s\n", Gsym[id].name, reglist[r]);
      if (op == A_POSTINC)
        fprintf(Outfile, "\tincb\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_POSTDEC)
        fprintf(Outfile, "\tdecb\t%s(\%%rip)\n", Gsym[id].name);
      break;
    case P_INT:
      if (op == A_PREINC)
        fprintf(Outfile, "\tincl\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_PREDEC)
        fprintf(Outfile, "\tdecl\t%s(\%%rip)\n", Gsym[id].name);
      fprintf(Outfile, "\tmovslq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
      if (op == A_POSTINC)
        fprintf(Outfile, "\tincl\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_POSTDEC)
        fprintf(Outfile, "\tdecl\t%s(\%%rip)\n", Gsym[id].name);
      break;
    case P_LONG:
    case P_CHARPTR:
    case P_INTPTR:
    case P_LONGPTR:
      if (op == A_PREINC)
        fprintf(Outfile, "\tincq\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_PREDEC)
        fprintf(Outfile, "\tdecq\t%s(\%%rip)\n", Gsym[id].name);
      fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
      if (op == A_POSTINC)
        fprintf(Outfile, "\tincq\t%s(\%%rip)\n", Gsym[id].name);
      if (op == A_POSTDEC)
        fprintf(Outfile, "\tdecq\t%s(\%%rip)\n", Gsym[id].name);
      break;
    default:
      fatald("Bad type in cgloadglob:", Gsym[id].type);
  }
  return (r);
}
```

我很确定我以后必须重写这个来执行`x= b[5]++`，但现在这就够了。毕竟，一步一个脚印是我对我们旅程的每一步所承诺的。

## 测试新功能

对于这一步，我不会详细介绍新的测试输入文件。它们是`tests`目录中的`input22.c`、`input23.c`和`input24.c`。你可以浏览它们并确认编译器可以正确编译它们：

```c
$ make test
...
input22.c: OK
input23.c: OK
input24.c: OK
```

## 总结与展望

就扩展编译器的功能而言，这部分旅程增加了很多功能，但我希望额外的概念复杂性是最小的。

我们添加了一组二元运算符，这是通过更新扫描器和更改运算符优先级表来完成的。

对于一元运算符，我们在 `prefix()` 函数中将它们手动添加到解析器中。

对于新的后缀运算符，我们将旧的函数调用和数组索引功能分离为一个新的 `postfix()` 函数，并使用它添加后缀运算符。在这里，我们确实需要担心一些左值和右值。我们还就添加什么 AST 节点，或者是否应该重新装饰一些现有的 AST 节点做出了一些设计决策。

代码生成最终相对简单，因为 x86-64 架构有实现我们所需操作的指令。然而，我们确实需要为某些操作设置一些特定的寄存器，或者执行指令组合来完成我们想要的操作。

棘手的操作是递增和递减操作。我已经编写了代码，让这些代码适用于普通变量，但我们稍后必须重新讨论。

在编译器编写之旅的下一部分，我将讨论局部变量。一旦我们能让这些工作起来，我们就可以将它们扩展到包括函数参数和参数。这将需要两个或更多步骤。