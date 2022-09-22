# Part 8：IF 语句

现在我们可以比较值了，是时候向我们的语言中添加 IF 语句了。首先，让我们看看 IF 语句的一般语法，以及它们是如何转换成汇编语言的。

## IF 语法

IF 语句的语法是：

```c
  if (condition is true) 
    perform this first block of code
  else
    perform this other block of code
```

那么，这通常是如何转换成汇编语言的呢？事实表明，我们进行相反的比较，如果相反的比较为真，则跳转 / 分支。

```c
       perform the opposite comparison
       jump to L1 if true
       perform the first block of code
       jump to L2
L1:
       perform the other block of code
L2:
   
```

其中 L1 和 L2 是汇编语言标记。

## 在我们的编译器中生成汇编

现在，我们基于比较输出代码来设置寄存器，例如

```c
   int x; x= 7 < 9;         From input04
```

变成

```assembly
        movq    $7, %r8
        movq    $9, %r9
        cmpq    %r9, %r8
        setl    %r9b        Set if less than 
        andq    $255,%r9
```

但是对于 IF 语句，我们需要进行相反的比较：

```c
   if (7 < 9) 
```

应该变成

```assembly
        movq    $7, %r8
        movq    $9, %r9
        cmpq    %r9, %r8
        jge     L1         Jump if greater then or equal to
        ....
L1:
```

所以，在这段旅程中，我已经实现了 IF 语句。由于这是一个工作项目（working project），作为旅程的一部分，我确实不得不撤销一些东西，并对它们进行重构。我将尝试涵盖这些变化以及沿途的补充内容。

## 新的 token 与悬空 else

我们的语言中需要一些新的 token。我还（目前）希望避免 [悬空 else](https://en.wikipedia.org/wiki/Dangling_else)（dangling else） 问题。为此，我修改了语法，以便所有语句组都围绕 '{' ... '}' 花括号；我把这样的分组称为“复合语句”。我们还需要 '(' ... ')' 括号保存 IF 表达式，加上关键字 'if' 和 'else'。因此，新的 token 是（在 `def.h` 中）：

```c
  T_LBRACE, T_RBRACE, T_LPAREN, T_RPAREN,
  // Keywords
  ..., T_IF, T_ELSE
```

## 扫描 token

单字符 token 应该是显而易见的，我不会给出扫描它们的代码。关键字也应该非常明显，但是我将给出 `scan.c` 中 ` keyword()` 的扫描代码：

```c
  switch (*s) {
    case 'e':
      if (!strcmp(s, "else"))
        return (T_ELSE);
      break;
    case 'i':
      if (!strcmp(s, "if"))
        return (T_IF);
      if (!strcmp(s, "int"))
        return (T_INT);
      break;
    case 'p':
      if (!strcmp(s, "print"))
        return (T_PRINT);
      break;
  }
```

## 新的 BNF 语法

我们的语法开始变得庞大，所以我稍微重写了一下：

```c
 compound_statement: '{' '}'          // empty, i.e. no statement
      |      '{' statement '}'
      |      '{' statement statements '}'
      ;

 statement: print_statement
      |     declaration
      |     assignment_statement
      |     if_statement
      ;

 print_statement: 'print' expression ';'  ;

 declaration: 'int' identifier ';'  ;

 assignment_statement: identifier '=' expression ';'   ;

 if_statement: if_head
      |        if_head 'else' compound_statement
      ;

 if_head: 'if' '(' true_false_expression ')' compound_statement  ;

 identifier: T_IDENT ;
```

我省略了 `true_false_expression` 的定义，但当我们再添加一些运算符时，我会把它加进去。

注意 IF 语句的语法：它要么是一个 if_head（没有 'else' 子句），要么是一个 `if_head` 后面跟着一个 'else' 和一个 `compound_statement`。

我已经将所有不同的语句类型分离出来，让它们有自己的非终结符名称。另外，前面的 `statements` 非终结符现在是 `compound_statement` 非终结符，这需要 '{' ... '}' 括起来。

这意味着头部的 `compound_statement` 被 '{' ... '}' 包围，'else' 关键字后面的任何 `compound_statement` 也是如此。因此，如果我们有嵌套的 `if` 语句，它们必须如下所示：

```c
  if (condition1 is true) {
    if (condition2 is true) {
      statements;
    } else {
      statements;
    }
  } else {
    statements;
  }
```

而且对于“if”和“each”属于哪一个也没有歧义。这就解决了悬空 else 问题。稍后，我将使 '{'…'}' 成为可选的。

## 解析复合语句

旧的 `void statements()` 函数现在是 `compound_statement()`，如下所示：

```c
// Parse a compound statement
// and return its AST
struct ASTnode *compound_statement(void) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  // Require a left curly bracket
  lbrace();

  while (1) {
    switch (Token.token) {
      case T_PRINT:
        tree = print_statement();
        break;
      case T_INT:
        var_declaration();
        tree = NULL;            // No AST generated here
        break;
      case T_IDENT:
        tree = assignment_statement();
        break;
      case T_IF:
        tree = if_statement();
        break;
    case T_RBRACE:
        // When we hit a right curly bracket,
        // skip past it and return the AST
        rbrace();
        return (left);
      default:
        fatald("Syntax error, token", Token.token);
    }

    // For each new tree, either save it in left
    // if left is empty, or glue the left and the
    // new tree together
    if (tree) {
      if (left == NULL)
        left = tree;
      else
        left = mkastnode(A_GLUE, left, NULL, tree, 0);
    }
  }
```

首先，请注意，代码强制解析器用 `lbrace()` 匹配复合语句开头的 '{ '，只有在用 `rbrace()` 匹配了结尾的 '}' 后，我们才能退出。

其次，注意 `print_statement()`、`assignment_statement()` 和 `if_statement()` 都返回一个 AST 树，`compound_statement()` 也是如此。在我们的旧代码中，`print_statement()` 本身调用 `genAST()` 来计算表达式，然后调用 `genprintint()`。类似地，`assignment_statement()` 也调用 `genAST()` 进行赋值。

这意味着我们在这里有 AST 树，在那里有其他树。只生成单个 AST 树，然后调用 `genAST()` 一次为其生成汇编代码是有意义的。

这不是强制性的。例如，SubC 只为表达式生成 ASTs。对于该语言的结构部分，如语句，SubC 对代码生成器进行特定的调用，就像我在以前版本的编译器中所做的那样。

现在，我决定用解析器为整个输入生成一个 AST 树。一旦输入被解析，就可以从一个AST树中生成汇编输出。

稍后，我可能会为每个函数生成一个 AST 树。后面再说。

## 解析 IF 语法

因为我们是递归下降解析器，所以解析 IF 语句还不算太差：

```c
// Parse an IF statement including
// any optional ELSE clause
// and return its AST
struct ASTnode *if_statement(void) {
  struct ASTnode *condAST, *trueAST, *falseAST = NULL;

  // Ensure we have 'if' '('
  match(T_IF, "if");
  lparen();

  // Parse the following expression
  // and the ')' following. Ensure
  // the tree's operation is a comparison.
  condAST = binexpr(0);

  if (condAST->op < A_EQ || condAST->op > A_GE)
    fatal("Bad comparison operator");
  rparen();

  // Get the AST for the compound statement
  trueAST = compound_statement();

  // If we have an 'else', skip it
  // and get the AST for the compound statement
  if (Token.token == T_ELSE) {
    scan(&Token);
    falseAST = compound_statement();
  }
  // Build and return the AST for this statement
  return (mkastnode(A_IF, condAST, trueAST, falseAST, 0));
}
```

现在，我不想处理像 `if (x-2)` 这样的输入，所以我限制了 `binexpr()` 的二元表达式的根节点的类型，它是六个比较运算符 A_EQ、A_NE、A_LT、A_GT、A_LE 或 A_GE 之一。

## 第三个孩子

我差点没解释清楚就偷偷带了些东西给你。在 `if_statement()` 的最后一行，我用以下代码构建了一个 AST 节点：

```c
   mkastnode(A_IF, condAST, trueAST, falseAST, 0);
```

那是三棵 AST 子树！这是怎么回事？如你所见，IF 语句有三个子语句：

- 计算条件的子树
- 紧随其后的复合语句
- 'else' 关键字后的可选复合语句

所以我们现在需要一个带有三个子节点的 AST 节点结构（在 `defs.h` 中）：

```c
// AST node types.
enum {
  ...
  A_GLUE, A_IF
};

// Abstract Syntax Tree structure
struct ASTnode {
  int op;                       // "Operation" to be performed on this tree
  struct ASTnode *left;         // Left, middle and right child trees
  struct ASTnode *mid;
  struct ASTnode *right;
  union {
    int intvalue;               // For A_INTLIT, the integer value
    int id;                     // For A_IDENT, the symbol slot number
  } v;
};
```

因此，A_IF 树看起来像这样：

```
                      IF
                    / |  \
                   /  |   \
                  /   |    \
                 /    |     \
                /     |      \
               /      |       \
      condition   statements   statements
```

## 粘合 AST 节点

还有一个新的 A_GLUE AST 节点类型。这是做什么用的？我们现在构建了一个包含大量语句的 AST 树，因此我们需要一种方法将它们粘合在一起。

回顾 `compound_statement()` 循环代码的结尾：

```c
  if (left != NULL)
    left = mkastnode(A_GLUE, left, NULL, tree, 0);
```

每当我们得到一个新的子树，我们就把它粘在现有的树上。因此，对于这一系列的语句：

```c
    stmt1;
    stmt2;
    stmt3;
    stmt4;
```

我们最终会得到：

```c
             A_GLUE
              /  \
          A_GLUE stmt4
            /  \
        A_GLUE stmt3
          /  \
      stmt1  stmt2
```

而且，当我们从左到右深度优先遍历树时，仍然会以正确的顺序生成汇编代码。

## 通用代码生成器

既然 AST 节点有多个子节点，我们的通用代码生成器将变得更加复杂一些。另外，对于比较运算符，我们需要知道是作为 if 语句的一部分（跳转到相反的比较）还是作为普通表达式（在普通比较时将寄存器设置为 1 或 0）进行比较。

为此，我修改了 `getAST()`，以便我们可以传入父 AST 节点操作:

```c
// Given an AST, the register (if any) that holds
// the previous rvalue, and the AST op of the parent,
// generate assembly code recursively.
// Return the register id with the tree's final value
int genAST(struct ASTnode *n, int reg, int parentASTop) {
   ...
}
```

## 处理特定的 AST 节点

`genAST()` 中的代码现在必须处理特定的 AST 节点：

```c
  // We now have specific AST node handling at the top
  switch (n->op) {
    case A_IF:
      return (genIFAST(n));
    case A_GLUE:
      // Do each child statement, and free the
      // registers after each child
      genAST(n->left, NOREG, n->op);
      genfreeregs();
      genAST(n->right, NOREG, n->op);
      genfreeregs();
      return (NOREG);
  }
```

如果不返回，则继续执行常规的二进制运算符 AST 节点，但有一个例外：比较节点：

```c
    case A_EQ:
    case A_NE:
    case A_LT:
    case A_GT:
    case A_LE:
    case A_GE:
      // If the parent AST node is an A_IF, generate a compare
      // followed by a jump. Otherwise, compare registers and
      // set one to 1 or 0 based on the comparison.
      if (parentASTop == A_IF)
        return (cgcompare_and_jump(n->op, leftreg, rightreg, reg));
      else
        return (cgcompare_and_set(n->op, leftreg, rightreg));
```

我将在下面介绍新函数 `cgcompare_and_jump()` 和 `cgcompate_and_set()`。

##  生成 IF 汇编代码

我们用一个特定的函数以及一个生成新标签号的函数来处理 A_IF AST 节点，：

```c
// Generate and return a new label number
static int label(void) {
  static int id = 1;
  return (id++);
}

// Generate the code for an IF statement
// and an optional ELSE clause
static int genIFAST(struct ASTnode *n) {
  int Lfalse, Lend;

  // Generate two labels: one for the
  // false compound statement, and one
  // for the end of the overall IF statement.
  // When there is no ELSE clause, Lfalse _is_
  // the ending label!
  Lfalse = label();
  if (n->right)
    Lend = label();

  // Generate the condition code followed
  // by a zero jump to the false label.
  // We cheat by sending the Lfalse label as a register.
  genAST(n->left, Lfalse, n->op);
  genfreeregs();

  // Generate the true compound statement
  genAST(n->mid, NOREG, n->op);
  genfreeregs();

  // If there is an optional ELSE clause,
  // generate the jump to skip to the end
  if (n->right)
    cgjump(Lend);

  // Now the false label
  cglabel(Lfalse);

  // Optional ELSE clause: generate the
  // false compound statement and the
  // end label
  if (n->right) {
    genAST(n->right, NOREG, n->op);
    genfreeregs();
    cglabel(Lend);
  }

  return (NOREG);
}
```

实际上，代码正在执行以下操作：

```c
  genAST(n->left, Lfalse, n->op);       // Condition and jump to Lfalse
  genAST(n->mid, NOREG, n->op);         // Statements after 'if'
  cgjump(Lend);                         // Jump to Lend
  cglabel(Lfalse);                      // Lfalse: label
  genAST(n->right, NOREG, n->op);       // Statements after 'else'
  cglabel(Lend);                        // Lend: label
```

## x86-64 代码生成函数

所以我们现在有了一些新的 x86-64 代码生成函数。其中一些函数取代了我们在之前创建的 6 个 cgXXX() 比较函数。

对于普通的比较函数，我们现在传入 AST 操作以选择相关的 x86-64 集合指令：

```c
// List of comparison instructions,
// in AST order: A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE
static char *cmplist[] =
  { "sete", "setne", "setl", "setg", "setle", "setge" };

// Compare two registers and set if true.
int cgcompare_and_set(int ASTop, int r1, int r2) {

  // Check the range of the AST operation
  if (ASTop < A_EQ || ASTop > A_GE)
    fatal("Bad ASTop in cgcompare_and_set()");

  fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  fprintf(Outfile, "\t%s\t%s\n", cmplist[ASTop - A_EQ], breglist[r2]);
  fprintf(Outfile, "\tmovzbq\t%s, %s\n", breglist[r2], reglist[r2]);
  free_register(r1);
  return (r2);
}
```

我还发现了一个 x86-64 指令 `movzbq`，它从一个寄存器中移动最低字节，并将其扩展到 64 位寄存器中。我现在用它代替了旧代码中的 `and $255`。

我们需要一个函数来生成一个标签并跳转到它：

```c
// Generate a label
void cglabel(int l) {
  fprintf(Outfile, "L%d:\n", l);
}

// Generate a jump to a label
void cgjump(int l) {
  fprintf(Outfile, "\tjmp\tL%d\n", l);
}
```

最后，我们需要一个函数来进行比较并基于相反的比较进行跳转。因此，使用 AST 比较节点类型，我们进行相反的比较：

```c
// List of inverted jump instructions,
// in AST order: A_EQ, A_NE, A_LT, A_GT, A_LE, A_GE
static char *invcmplist[] = { "jne", "je", "jge", "jle", "jg", "jl" };

// Compare two registers and jump if false.
int cgcompare_and_jump(int ASTop, int r1, int r2, int label) {

  // Check the range of the AST operation
  if (ASTop < A_EQ || ASTop > A_GE)
    fatal("Bad ASTop in cgcompare_and_set()");

  fprintf(Outfile, "\tcmpq\t%s, %s\n", reglist[r2], reglist[r1]);
  fprintf(Outfile, "\t%s\tL%d\n", invcmplist[ASTop - A_EQ], label);
  freeall_registers();
  return (NOREG);
}
```

## 测试 IF 语句

执行 `make test`，编译 `input05` 文件：

```c
{
  int i; int j;
  i=6; j=12;
  if (i < j) {
    print i;
  } else {
    print j;
  }
}
```

下面是生成的汇编输出：

```c
        movq    $6, %r8
        movq    %r8, i(%rip)    # i=6;
        movq    $12, %r8
        movq    %r8, j(%rip)    # j=12;
        movq    i(%rip), %r8
        movq    j(%rip), %r9
        cmpq    %r9, %r8        # Compare %r8-%r9, i.e. i-j
        jge     L1              # Jump to L1 if i >= j
        movq    i(%rip), %r8
        movq    %r8, %rdi       # print i;
        call    printint
        jmp     L2              # Skip the else code
L1:
        movq    j(%rip), %r8
        movq    %r8, %rdi       # print j;
        call    printint
L2:
```

当然，`make test` 显示：

```c
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c
./comp1 input05
cc -o out out.s
./out
6                   # As 6 is less than 12
```

## 总结与展望

我们已经用 IF 语句在语言中添加了第一个控制结构。在这个过程中，我不得不重写一些现有的东西，鉴于我头脑中没有一个完整的架构计划，我可能不得不在未来重写更多的东西。

这一部分的问题是，我们必须为 IF 决策执行与普通比较运算符相反的比较。我的解决方案是通知每个 AST 节点其父节点的节点类型；比较节点现在可以看到父节点是否是 A_IF 节点。

我知道 Nils Holm 在实现 SubC 时选择了一种不同的方法，所以你应该看看他的代码，看看对同一问题的不同解决方案。

在编译器编写旅程的下一部分，我们将添加另一个控制结构：WHILE 循环。
