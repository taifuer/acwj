# Part 12：类型，第一部分

我刚刚开始向我们的编译器添加类型。现在，我应该警告你，这对我来说是新的，因为在我以前的编译器中只有 `int` 类型。我抑制了查看 SubC 源代码寻找灵感的冲动。因此，我正在独自努力，当我处理更大的涉及类型的问题时，我可能不得不重写一些代码。

## 现在什么类型？

我将从 `char` 类型和 `int` 类型的全局变量开始。我们已经为函数添加了 `void` 关键字。下一步，我将添加函数返回值。现在，`void` 存在，但我没有完全处理它。

显然，`char` 类型的值范围比 int 类型有限得多。和 SubC 一样，我将使用范围 0 到 255 用于 `char` s，有符号值的范围用于 `int` s。

这意味着我们可以将 `char` 值扩大为 `int` 值，但如果开发人员试图将 `int` 值缩小为 `char` 值，我们必须警告他们。

## 新关键字和标记

只有新的 `char` 关键字和 T_CHAR 标记。这里没什么令人兴奋的。

## 表达式类型

从现在开始，每个表达式都有一个类型。这包括：

- 整型字面量，例如 56 是一个 `int`
- 数学表达式，例如 45 - 12 是一个 `int`
- 变量，例如，如果我们声明 `x` 为 `char` 类型，那么它的右值是一个 `char`

当我们计算每个表达式时，我们必须跟踪它的类型，以确保我们可以根据需要扩大它（`char` 可以转 `int`），或者在必要时拒绝缩小它（`int` 不能转 `char`）。

在 SubC 编译器中，Nils 创建了一个单左值结构。递归解析器中传递了一个指向这个单一结构的指针，以跟踪任何表达式在解析时的类型。

我采取了不同的策略。我已经修改了抽象语法树节点，使其具有一个类型字段，该字段保存了此时树的类型。在 `def.h` 中，这里是目前为止我创建的类型:

```c
// Primitive types
enum {
  P_NONE, P_VOID, P_CHAR, P_INT
};
```

我将它们称为原始类型，就像 Nils 在 SubC 中所做的那样，因为我想不出更好的名称来称呼它们。数据类型，可能吗？P_NONE 值表示 AST 节点不表示表达式，也没有类型。一个例子是将语句粘合在一起的 A_GLUE 节点类型：一旦生成左边的语句，就没有类型可言了。

如果你查看 `tree.c`，你将会看到构建 AST 节点的函数已经被修改为也分配给新 AST 节点结构中的 `type` 字段（在 `defs.h` 中）：

```c
struct ASTnode {
  int op;                       // "Operation" to be performed on this tree
  int type;                     // Type of any expression this tree generates
  ...
};
```

## 变量声明及其类型

我们现在至少有两种方法来声明全局变量：

```c
  int x; char y;
```

是的，我们需要分析这个。但是首先，我们如何记录每个变量的类型？我们需要修改 `symtable` 结构。我还添加了我将在未来使用的符号的“结构类型”的细节（在 `defs.h` 中）：

```c
// Structural types
enum {
  S_VARIABLE, S_FUNCTION
};

// Symbol table structure
struct symtable {
  char *name;                   // Name of a symbol
  int type;                     // Primitive type for the symbol
  int stype;                    // Structural type for the symbol
};
```

`sym.c` 中的 `newglob()` 中有新代码来初始化这些新字段：

```c
int addglob(char *name, int type, int stype) {
  ...
  Gsym[y].type = type;
  Gsym[y].stype = stype;
  return (y);
}
```

## 解析变量声明

是时候将类型的解析与变量本身的解析分开了。因此，在 `decl.c` 中，我们现在有：

```c
// Parse the current token and
// return a primitive type enum value
int parse_type(int t) {
  if (t == T_CHAR) return (P_CHAR);
  if (t == T_INT)  return (P_INT);
  if (t == T_VOID) return (P_VOID);
  fatald("Illegal type, token", t);
}

// Parse the declaration of a variable
void var_declaration(void) {
  int id, type;

  // Get the type of the variable, then the identifier
  type = parse_type(Token.token);
  scan(&Token);
  ident();
  id = addglob(Text, type, S_VARIABLE);
  genglobsym(id);
  semi();
}
```

## 处理表达式类型

以上都是容易做到的部分！我们现在有：

- 一组三种类型：`char`、`int` 和 `void`，
- 解析变量声明以查找其类型，
- 捕获符号表中每个变量的类型，以及
- 每个 AST 节点中表达式类型的存储

现在我们需要在构建的 AST 节点中实际填充类型。然后，我们必须决定何时扩大类型和/或拒绝类型冲突。让我们继续工作吧！

## 解析主要终结符

我们将从解析整型字面量和变量标识符开始。其中一个问题是，我们希望能够做到：

```c
  char j; j= 2;
```

但是如果我们将 2 标记为 P_INT，那么当我们试图将其存储在 P_CHAR `j`变量中时，就不能缩小该值。现在，我添加了一些语义代码来保持小的整型字面量 P_CHARs：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
    case T_INTLIT:
      // For an INTLIT token, make a leaf AST node for it.
      // Make it a P_CHAR if it's within the P_CHAR range
      if ((Token.intvalue) >= 0 && (Token.intvalue < 256))
        n = mkastleaf(A_INTLIT, P_CHAR, Token.intvalue);
      else
        n = mkastleaf(A_INTLIT, P_INT, Token.intvalue);
      break;

    case T_IDENT:
      // Check that this identifier exists
      id = findglob(Text);
      if (id == -1)
        fatals("Unknown variable", Text);

      // Make a leaf AST node for it
      n = mkastleaf(A_IDENT, Gsym[id].type, id);
      break;

    default:
      fatald("Syntax error, token", Token.token);
  }

  // Scan in the next token and return the leaf node
  scan(&Token);
  return (n);
}
```

还要注意，对于标识符，我们可以很容易地从全局符号表中获得其类型详细信息。

## 构建二元表达式：比较类型

当我们用二元数学运算符构建数学表达式时，我们将有一个来自左孩子的类型和一个来自右孩子的类型。在这里，如果两种类型不兼容，我们将不得不加宽、不做任何事情或拒绝表达式。

现在，我有了一个新的文件 `types.c`，它有一个比较两边类型的函数。代码如下：

```c
// Given two primitive types, return true if they are compatible,
// false otherwise. Also return either zero or an A_WIDEN
// operation if one has to be widened to match the other.
// If onlyright is true, only widen left to right.
int type_compatible(int *left, int *right, int onlyright) {

  // Voids not compatible with anything
  if ((*left == P_VOID) || (*right == P_VOID)) return (0);

  // Same types, they are compatible
  if (*left == *right) { *left = *right = 0; return (1);
  }

  // Widen P_CHARs to P_INTs as required
  if ((*left == P_CHAR) && (*right == P_INT)) {
    *left = A_WIDEN; *right = 0; return (1);
  }
  if ((*left == P_INT) && (*right == P_CHAR)) {
    if (onlyright) return (0);
    *left = 0; *right = A_WIDEN; return (1);
  }
  // Anything remaining is compatible
  *left = *right = 0;
  return (1);
}
```

这里发生了不少事情。首先，如果两种类型都相同，我们可以简单地返回 True。带有 P_VOID 的任何内容都不能与其他类型混合。

如果一侧是 P_CHAR，另一侧是 P_INT，我们可以将结果加宽为 P_INT。我这样做的方法是修改输入的类型，并将其替换为零（什么也不做）或新的 AST 节点类型 A_WIDEN。这意味着：将更窄的孩子的值加宽到与更宽的孩子的值一样宽。我们很快就会看到它的运行。

有一个额外的参数是 `onlyright`。当我们到达 A_ASSIGN AST 节点时，我会使用这个方法，在这里我们将左边子节点的表达式赋给右边的变量 lvalue。如果设置了此选项，则不要将 P_INT 表达式转换到 P_CHAR 变量。

最后，现在，让任何其他类型对通过。

我想我可以保证，一旦引入数组和指针，就需要对其进行更改。我还希望能找到一种方法，使代码更简单、更优雅。但现在还行。

## 在表达式中使用 `type_compatible()`

在这个版本的编译器中，我在三个不同的地方使用了 `type_compatible()`。我们将从用二元运算符合并表达式开始。为此，我修改了 `expr.c` 中 `binexpr()` 的代码：

```c
    // Ensure the two types are compatible.
    lefttype = left->type;
    righttype = right->type;
    if (!type_compatible(&lefttype, &righttype, 0))
      fatal("Incompatible types");

    // Widen either side if required. type vars are A_WIDEN now
    if (lefttype)
      left = mkastunary(lefttype, right->type, left, 0);
    if (righttype)
      right = mkastunary(righttype, left->type, right, 0);

    // Join that sub-tree with ours. Convert the token
    // into an AST operation at the same time.
    left = mkastnode(arithop(tokentype), left->type, left, NULL, right, 0);
```

我们拒绝不兼容的类型。但是，如果 `type_compatible()` 返回非零的 `lefttype` 或 `righttype` 值，这些实际上是 A_WIDEN  值。我们可以使用它来构建一个以窄子节点为子节点的一元 AST 节点。当我们到达代码生成器时，它将知道这个子值必须加宽。

现在，我们还需要在哪里扩展表达式值？

## 使用 `type_compatible()` 打印表达式

当我们使用 `print` 关键字时，我们需要一个 `int` 表达式来打印它。所以我们需要改变 `stmt.c` 中的 `print_statement()`：

```c
static struct ASTnode *print_statement(void) {
  struct ASTnode *tree;
  int lefttype, righttype;
  int reg;

  ...
  // Parse the following expression
  tree = binexpr(0);

  // Ensure the two types are compatible.
  lefttype = P_INT; righttype = tree->type;
  if (!type_compatible(&lefttype, &righttype, 0))
    fatal("Incompatible types");

  // Widen the tree if required. 
  if (righttype) tree = mkastunary(righttype, P_INT, tree, 0);
```

## 使用 `type_compatible()` 给变量赋值

这是我们需要检查类型的最后一个地方。当赋值给一个变量时，我们需要确保可以扩大右边的表达式。我们必须拒绝将宽类型存储到窄变量中的任何尝试。下面是 `stmt.c` 中 `assignment_statement()` 的新代码：

```c
static struct ASTnode *assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int lefttype, righttype;
  int id;

  ...
  // Make an lvalue node for the variable
  right = mkastleaf(A_LVIDENT, Gsym[id].type, id);

  // Parse the following expression
  left = binexpr(0);

  // Ensure the two types are compatible.
  lefttype = left->type;
  righttype = right->type;
  if (!type_compatible(&lefttype, &righttype, 1))  // Note the 1
    fatal("Incompatible types");

  // Widen the left if required.
  if (lefttype)
    left = mkastunary(lefttype, right->type, left, 0);
```

注意 `type_compatible()` 调用末尾的1。这强制了不能将宽值保存到窄变量的语义。

考虑到以上所有情况，我们现在可以解析一些类型并强制执行一些合理的语言语义：在可能的情况下扩大值，防止类型缩小和防止不适当的类型冲突。现在我们转向代码生成方面的内容。

## x86-64 代码生成的变化

我们的汇编输出是基于寄存器的，本质上它们的大小是固定的。我们能影响的是：

- 存储变量的内存位置的大小，以及
- 使用多少寄存器来保存数据，例如一个字节用于字符，八个字节用于 64 位整数。

我将从 `cg.c` 中特定于 x86-64 的代码开始，然后展示如何在 `gen.c` 中的通用代码生成器中使用它。

让我们从生成存储变量的代码开始。

```c
// Generate a global symbol
void cgglobsym(int id) {
  // Choose P_INT or P_CHAR
  if (Gsym[id].type == P_INT)
    fprintf(Outfile, "\t.comm\t%s,8,8\n", Gsym[id].name);
  else
    fprintf(Outfile, "\t.comm\t%s,1,1\n", Gsym[id].name);
}
```

我们从符号表中的变量槽中提取类型，并根据类型选择为其分配 1 或 8 个字节。现在我们需要将值加载到寄存器中：

```c
// Load a value from a variable into a register.
// Return the number of the register
int cgloadglob(int id) {
  // Get a new register
  int r = alloc_register();

  // Print out the code to initialise it: P_CHAR or P_INT
  if (Gsym[id].type == P_INT)
    fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
  else
    fprintf(Outfile, "\tmovzbq\t%s(\%%rip), %s\n", Gsym[id].name, reglist[r]);
  return (r);
```

`movq` 指令将八个字节移动到 8 字节寄存器中。movzbq 指令将 8 字节寄存器置零，然后将单个字节移到其中。这还隐式地将一个字节的值扩大到八个字节。我们的存储函数类似：

```c
// Store a register's value into a variable
int cgstorglob(int r, int id) {
  // Choose P_INT or P_CHAR
  if (Gsym[id].type == P_INT)
    fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], Gsym[id].name);
  else
    fprintf(Outfile, "\tmovb\t%s, %s(\%%rip)\n", breglist[r], Gsym[id].name);
  return (r);
}
```

这一次，我们必须使用寄存器的“字节”名称和 movb 指令来移动单个字节。

幸运的是，`cgloadglob()` 函数已经扩展了 P_CHAR 变量。这是新 `cgwiden()` 函数的代码：

```c
// Widen the value in the register from the old
// to the new type, and return a register with
// this new value
int cgwiden(int r, int oldtype, int newtype) {
  // Nothing to do
  return (r);
}
```

## 通用代码生成的更改

有了上述内容，`gen.c` 中的通用代码生成器只有几处改动:

- 对 `cgloadglob()` 和 `cgstorglob()` 的调用现在采用符号的槽号，而不是符号的名称。
- 类似地，`genglobsym()`现在接收符号的槽号，并将其传递给`cgglobsym()`

唯一的主要变化是处理新的 A_WIDEN AST 节点类型的代码。我们不需要这个节点（因为 `cgwiden()` 什么也不做），但它在这里用于其他硬件平台：

```
    case A_WIDEN:
      // Widen the child's type to the parent's type
      return (cgwiden(leftreg, n->left->type, n->type));
```

## 测试新的类型

这是我的测试输入文件，`tests/input10`：

```c
void main()
{
  int i; char j;

  j= 20; print j;
  i= 10; print i;

  for (i= 1;   i <= 5; i= i + 1) { print i; }
  for (j= 253; j != 2; j= j + 1) { print j; }
}
```

我检查了我们可以从 `char` 和 `int` 类型赋值和打印。我还验证了，对于 `char` 变量，我们将在值序列中溢出：253，254，255，0，1，2 等等。

```
$ make test
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
   stmt.c sym.c tree.c types.c
./comp1 tests/input10
cc -o out out.s
./out
20
10
1
2
3
4
5
253
254
255
0
1
```

让我们看看生成的一些汇编代码：

```assembly
        .comm   i,8,8                   # Eight byte i storage
        .comm   j,1,1                   # One   byte j storage
        ...
        movq    $20, %r8
        movb    %r8b, j(%rip)           # j= 20
        movzbq  j(%rip), %r8
        movq    %r8, %rdi               # print j
        call    printint

        movq    $253, %r8
        movb    %r8b, j(%rip)           # j= 253
L3:
        movzbq  j(%rip), %r8
        movq    $2, %r9
        cmpq    %r9, %r8                # while j != 2
        je      L4
        movzbq  j(%rip), %r8
        movq    %r8, %rdi               # print j
        call    printint
        movzbq  j(%rip), %r8
        movq    $1, %r9                 # j= j + 1
        addq    %r8, %r9
        movb    %r9b, j(%rip)
        jmp     L3
```

仍然不是最优雅的汇编代码，但它确实有效。此外， `$ make test` 确认所有前面的代码示例仍然有效。

## 总结与展望

在编译器编写旅程的下一部分，我们将添加带有一个参数的函数调用，并从函数返回值。