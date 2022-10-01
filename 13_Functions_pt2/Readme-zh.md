# Part 13：函数，第 2 部分

在我们编译器编写旅程的这一部分，我想添加调用函数和返回值的能力。具体来说：

- 定义一个函数，我们已经有了，
- 调用一个目前无法使用的单值函数，
- 从函数中返回值，
- 将函数调用用作语句和表达式，并且
- 确保 void 函数永远不会返回值，而非 void 函数必须返回值。

我刚把它修好。我发现我大部分时间都在和类型打交道。所以，继续写吧。

## 新关键字和标记

到目前为止，我一直在编译器中使用 8 字节（64 位）的 `int`，但我意识到 GCC 将 `int` 视为 4 字节（32 位）宽。因此，我决定引入 `long` 类型。所以现在：

- `char`是 1 个字节宽
- `int` 是 4 个字节（32 位）宽
- `long` 是 8 字节（64 位）宽

我们还需要 'return' 功能，因此我们有了新的关键字 'long' 和 'return'，以及对应的标记 T_LONG 和T_RETURN。

## 解析函数调用

目前，我用于函数调用的 BNF 语法是：

```
  function_call: identifier '(' expression ')'   ;
```

函数的名称后面有一对括号。在括号内必须有一个参数。我希望它既被用作表达式，也被用作独立的声明。

因此，我们将从函数调用解析器 `expr.c` 中的 `funccall()` 开始。当我们被调用时，标识符已经被扫描进来，函数的名称在  `Text`  全局变量中：

```c
// Parse a function call with a single expression
// argument and return its AST
struct ASTnode *funccall(void) {
  struct ASTnode *tree;
  int id;

  // Check that the identifier has been defined,
  // then make a leaf node for it. XXX Add structural type test
  if ((id = findglob(Text)) == -1) {
    fatals("Undeclared function", Text);
  }
  // Get the '('
  lparen();

  // Parse the following expression
  tree = binexpr(0);

  // Build the function call AST node. Store the
  // function's return type as this node's type.
  // Also record the function's symbol-id
  tree = mkastunary(A_FUNCCALL, Gsym[id].type, tree, id);

  // Get the ')'
  rparen();
  return (tree);
}
```

我留下了一个提醒注释：添加结构类型检验。当声明函数或变量时，符号表将分别用结构类型S_FUNCTION 和 S_VARIABLE 标记。我应该在这里添加代码来确认标识符真的是一个 S_FUNCTION。

我们构建一个新的一元 AST 节点 A_FUNCCALL。子表达式是作为参数传递的唯一表达式。我们将函数的 symbol-id 存储在节点中，并记录函数的返回类型。

## 但是我再也不想要那个标记了！

有一个解析问题。我们必须区分：

```c
   x= fred + jim;
   x= fred(5) + jim;
```

我们需要向前看一个标记，看看是否有一个 '('。如果有，我们有一个函数调用。但是这样做，我们就失去了现有的标记。为了解决这个问题，我修改了扫描器，以便我们可以放回一个不需要的标记：当我们获得下一个标记而不是一个全新的标记时，它将被返回。`scan.c` 中的新代码是：

```c
// A pointer to a rejected token
static struct token *Rejtoken = NULL;

// Reject the token that we just scanned
void reject_token(struct token *t) {
  if (Rejtoken != NULL)
    fatal("Can't reject token twice");
  Rejtoken = t;
}

// Scan and return the next token found in the input.
// Return 1 if token valid, 0 if no tokens left.
int scan(struct token *t) {
  int c, tokentype;

  // If we have any rejected token, return it
  if (Rejtoken != NULL) {
    t = Rejtoken;
    Rejtoken = NULL;
    return (1);
  }

  // Continue on with the normal scanning
  ...
}
```

## 将函数作为表达式调用

现在我们来看看，在 `express.c` 中，我们需要区分变量名和函数调用：它在 `primary()` 中。新代码是：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
    ...
    case T_IDENT:
      // This could be a variable or a function call.
      // Scan in the next token to find out
      scan(&Token);

      // It's a '(', so a function call
      if (Token.token == T_LPAREN)
        return (funccall());

      // Not a function call, so reject the new token
      reject_token(&Token);

      // Continue on with normal variable parsing
      ...
}
```

## 将函数作为申明调用

当我们试图调用一个函数作为一个申明时，我们有本质上相同的问题。在这里，我们必须区分：

```c
  fred = 2;
  fred(18);
```

因此，`stmt.c` 中的新语句代码类似于上面的代码：

```c
// Parse an assignment statement and return its AST
static struct ASTnode *assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int lefttype, righttype;
  int id;

  // Ensure we have an identifier
  ident();

  // This could be a variable or a function call.
  // If next token is '(', it's a function call
  if (Token.token == T_LPAREN)
    return (funccall());

  // Not a function call, on with an assignment then!
  ...
}
```

我们可以在这里不拒绝“不需要的”标记，因为接下来必须有一个 '=' 或一个 '('：我们可以在知道这是真的的情况下编写解析器代码。

## 解析 return 语句

在 BNF 中，我们的返回语句是这样的：

```c
  return_statement: 'return' '(' expression ')'  ;
```

解析很简单：'return'，'('，调用 `binexpr()`， ')'，完成！更困难的是检查类型，以及是否允许我们返回。

当我们得到 return 语句时，我们需要知道我们实际上在哪个函数中。我在 `data.h` 中添加了一个全局变量：

```c
extern_ int Functionid;         // Symbol id of the current function
```

而这是在 `decl.c` 中的 `function_declaration()` 中设置的：

```c
struct ASTnode *function_declaration(void) {
  ...
  // Add the function to the symbol table
  // and set the Functionid global
  nameslot = addglob(Text, type, S_FUNCTION, endlabel);
  Functionid = nameslot;
  ...
}
```

通过在每次输入函数声明时设置 `Functionid`，我们可以重新解析和检查 return 语句的语义。新代码是 `stmt.c` 中的 `return_statement()`：

```c
// Parse a return statement and return its AST
static struct ASTnode *return_statement(void) {
  struct ASTnode *tree;
  int returntype, functype;

  // Can't return a value if function returns P_VOID
  if (Gsym[Functionid].type == P_VOID)
    fatal("Can't return from a void function");

  // Ensure we have 'return' '('
  match(T_RETURN, "return");
  lparen();

  // Parse the following expression
  tree = binexpr(0);

  // Ensure this is compatible with the function's type
  returntype = tree->type;
  functype = Gsym[Functionid].type;
  if (!type_compatible(&returntype, &functype, 1))
    fatal("Incompatible types");

  // Widen the left if required.
  if (returntype)
    tree = mkastunary(returntype, functype, tree, 0);

  // Add on the A_RETURN node
  tree = mkastunary(A_RETURN, P_NONE, tree, 0);

  // Get the ')'
  rparen();
  return (tree);
}
```

我们有一个新的 A_RETURN AST 节点，它返回子树中的表达式。我们使用 `type_compatible()` 来确保表达式匹配返回类型，并在需要时扩展它。

最后，我们看看这个函数是否真的被声明为 `void`。如果是，我们就不能在这个函数中执行 `return` 语句。

## 重访类型

在旅程的上一部分，我介绍了 `type_compatible()`，并说我想重构它。既然我已经添加了 `long` 类型，就有必要这样做了。这是 `types.c` 中的新版本，你可能想重温一下上一部分的注释。

```c
// Given two primitive types,
// return true if they are compatible,
// false otherwise. Also return either
// zero or an A_WIDEN operation if one
// has to be widened to match the other.
// If onlyright is true, only widen left to right.
int type_compatible(int *left, int *right, int onlyright) {
  int leftsize, rightsize;

  // Same types, they are compatible
  if (*left == *right) { *left = *right = 0; return (1); }
  // Get the sizes for each type
  leftsize = genprimsize(*left);
  rightsize = genprimsize(*right);

  // Types with zero size are not
  // not compatible with anything
  if ((leftsize == 0) || (rightsize == 0)) return (0);

  // Widen types as required
  if (leftsize < rightsize) { *left = A_WIDEN; *right = 0; return (1);
  }
  if (rightsize < leftsize) {
    if (onlyright) return (0);
    *left = 0; *right = A_WIDEN; return (1);
  }
  // Anything remaining is the same size
  // and thus compatible
  *left = *right = 0;
  return (1);
}
```

我现在在通用代码生成器中调用 `genprimsize()`，该生成器在 `cg.c` 中调用 `cgprimsize()` 来获得各种类型的大小：

```c
// Array of type sizes in P_XXX order.
// 0 means no size. P_NONE, P_VOID, P_CHAR, P_INT, P_LONG
static int psize[] = { 0,       0,      1,     4,     8 };

// Given a P_XXX type value, return the
// size of a primitive type in bytes.
int cgprimsize(int type) {
  // Check the type is valid
  if (type < P_NONE || type > P_LONG)
    fatal("Bad type in cgprimsize()");
  return (psize[type]);
}
```

这使得类型大小取决于平台，其他平台可以选择不同的类型大小。这可能意味着我将 P_INTLIT 标记为 `char` 而不是 `int` 的代码需要重构：

```c
  if ((Token.intvalue) >= 0 && (Token.intvalue < 256))
```

## 确保非 void 函数返回一个值

我们刚刚确保了 void 函数不能返回值。现在，我们如何确保非空函数总是返回一个值？为此，我们必须确保函数中的最后一条语句是 return 语句。

在 `decl.c` 的 `function_declaration()` 的底部，我现在有：

```c
  struct ASTnode *tree, *finalstmt;
  ...
  // If the function type isn't P_VOID, check that
  // the last AST operation in the compound statement
  // was a return statement
  if (type != P_VOID) {
    finalstmt = (tree->op == A_GLUE) ? tree->right : tree;
    if (finalstmt == NULL || finalstmt->op != A_RETURN)
      fatal("No return for function with non-void type");
  }
```

问题是，如果函数只有一个语句，那么就没有 A_GLUE AST 节点，树中只有一个复合语句的左子节点。

此时，我们可以：

- 声明一个函数，存储它的类型，并记录我们在那个函数中
- 使用单个参数进行函数调用（作为表达式或语句）
- 从非 void 函数返回（仅限），并强制非 void 函数中的最后一条语句是 return 语句
- 检查并扩展返回的表达式，以匹配函数的类型定义

我们的 AST 树现在有一个 `_RETURN` 和一个 `_FUNCCAL` 节点，用于返回语句和函数调用。现在让我们看看它们是如何生成汇编输出的。

## 为什么只有一个参数？

此时，您可能会问：为什么要使用单个函数参数，特别是当该参数对函数不可用时？

答案是我想把我们语言中的 `print x;`  语句换成一个真正的函数调用：`printint(x);`。为此，我们可以编译一个真正的 C 函数 `printint()`，并将其与编译器的输出相链接。

## 新的 AST 节点

在 `gen.c` 的 `genAST()` 中没有太多的新代码：

```c
    case A_RETURN:
      cgreturn(leftreg, Functionid);
      return (NOREG);
    case A_FUNCCALL:
      return (cgcall(leftreg, n->v.id));
```

A_RETURN 不返回值，因为它不是表达式。A_FUNCCALL 当然是一个表达式。

## x86-64 输出的更改

所有新的代码生成工作都在特定于平台的代码生成器 `cg.c` 中进行。让我们看看这个。

### 新类型

首先，我们现在有`char`、`int`和`long`，x86-64 要求我们为每种类型使用正确的寄存器名：

```
// List of available registers and their names.
static int freereg[4];
static char *reglist[4] = { "%r8", "%r9", "%r10", "%r11" };
static char *breglist[4] = { "%r8b", "%r9b", "%r10b", "%r11b" };
static char *dreglist[4] = { "%r8d", "%r9d", "%r10d", "%r11d" }
```

### 定义，加载和存储变量

变量现在有三种可能的类型。我们生成的代码需要反映这一点。以下是修改后的功能：

```c
// Generate a global symbol
void cgglobsym(int id) {
  int typesize;
  // Get the size of the type
  typesize = cgprimsize(Gsym[id].type);

  fprintf(Outfile, "\t.comm\t%s,%d,%d\n", Gsym[id].name, typesize, typesize);
}

// Load a value from a variable into a register.
// Return the number of the register
int cgloadglob(int id) {
  // Get a new register
  int r = alloc_register();

  // Print out the code to initialise it
  switch (Gsym[id].type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovzbq\t%s(\%%rip), %s\n", Gsym[id].name,
              reglist[r]);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovzbl\t%s(\%%rip), %s\n", Gsym[id].name,
              reglist[r]);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
      break;
    default:
      fatald("Bad type in cgloadglob:", Gsym[id].type);
  }
  return (r);
}

// Store a register's value into a variable
int cgstorglob(int r, int id) {
  switch (Gsym[id].type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovb\t%s, %s(\%%rip)\n", breglist[r],
              Gsym[id].name);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovl\t%s, %s(\%%rip)\n", dreglist[r],
              Gsym[id].name);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], Gsym[id].name);
      break;
    default:
      fatald("Bad type in cgloadglob:", Gsym[id].type);
  }
  return (r);
}
```

### 函数调用

要调用只有一个实参的函数，需要将带有实参值的寄存器复制到 `%rdi` 中。在返回时，我们需要将 `%rax` 的返回值复制到具有新值的寄存器中：

```c
// Call a function with one argument from the given register
// Return the register with the result
int cgcall(int r, int id) {
  // Get a new register
  int outr = alloc_register();
  fprintf(Outfile, "\tmovq\t%s, %%rdi\n", reglist[r]);
  fprintf(Outfile, "\tcall\t%s\n", Gsym[id].name);
  fprintf(Outfile, "\tmovq\t%%rax, %s\n", reglist[outr]);
  free_register(r);
  return (outr);
}
```

### 函数返回

要从函数执行的任何点返回函数，我们需要跳转到函数底部的标签。我在 `function_declaration()` 中添加了代码来创建一个标签并将其存储在符号表中。由于返回值留在 `%rax` 寄存器中，在跳转到结束标签之前，我们需要复制到这个寄存器中：

```c
// Generate code to return a value from a function
void cgreturn(int reg, int id) {
  // Generate code depending on the function's type
  switch (Gsym[id].type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovzbl\t%s, %%eax\n", breglist[reg]);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovl\t%s, %%eax\n", dreglist[reg]);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s, %%rax\n", reglist[reg]);
      break;
    default:
      fatald("Bad function type in cgreturn:", Gsym[id].type);
  }
  cgjump(Gsym[id].endlabel);
}
```

### 函数前导代码和后导代码的修改

前导代码没有变化，但之前我们在返回时将 `%rax` 设置为零。我们必须删除这段代码:

```c
// Print out a function postamble
void cgfuncpostamble(int id) {
  cglabel(Gsym[id].endlabel);
  fputs("\tpopq %rbp\n" "\tret\n", Outfile);
}
```

### 初始前导代码的修改

到目前为止，我一直在我们的汇编输出的开头手动插入一个汇编版本的 `printint()`。我们不再需要这个了，因为我们可以编译一个真正的 C 函数 `printint()` 并将它与编译器的输出相链接。

## 测试更改

有一个新的测试程序，`tests/input14`：

```
int fred() {
  return(20);
}

void main() {
  int result;
  printint(10);
  result= fred(15);
  printint(result);
  printint(fred(15)+10);
  return(0);
}
```

我们首先打印 10，然后调用返回 20 的 `fred()` 并打印出来。最后，我们再次调用 `fred()`，将其返回值与 10 相加，并打印出 30。这演示了使用单个值的函数调用，以及函数返回。以下是测试结果:

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
    stmt.c sym.c tree.c types.c
./comp1 tests/input14
cc -o out out.s lib/printint.c
./out; true
10
20
30
```

注意，我们用 `lib/printint.c` 链接我们的汇编输出：

```c
#include <stdio.h>
void printint(long x) {
  printf("%ld\n", x);
}
```

## 现在几乎是 C 了

有了这个改变，我们可以这样做：

```
$ cat lib/printint.c tests/input14 > input14.c
$ cc -o out input14.c
$ ./out 
10
20
30
```

换句话说，我们的语言是 C 语言的一个子集，我们可以用其他 C 函数来编译它，以获得一个可执行文件。太棒了。

## 总结与展望

我们刚刚添加了一个简单的函数调用版本，函数返回加上一个新的数据类型。正如我所料，这不是小事，但我认为这些变化大多是合理的。

在编译器编写旅程的下一部分，我们将把编译器移植到一个新的硬件平台上，即 Raspberry Pi 上的ARM CPU。