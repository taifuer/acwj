# Part 6：变量

我刚刚完成了向编译器添加全局变量的工作，正如我所怀疑的那样，这需要大量的工作。而且，编译器中的几乎每个文件都在这个过程中被修改。所以旅程的这一部分会很长。

## 我们想从变量中得到什么？

我们希望能够：

- 声明变量

- 使用变量获取存储的值

- 赋值给变量

这是 `input02`，它将是我们的测试程序：

```c
int fred;
int jim;
fred= 5;
jim= 12;
print fred + jim;
```

最明显的变化是语法现在在表达式中有变量声明、赋值语句和变量名。然而，在开始之前，让我们看看如何实现变量。

## 符号表

每个编译器都需要一个[符号表](https://en.wikipedia.org/wiki/Symbol_table)。稍后，我们将保存更多的全局变量。但是现在，表中条目的结构如下（来自 `def.h`）：

```c
// Symbol table structure
struct symtable {
  char *name;                   // Name of a symbol
};
```

我们在 `data.h` 中有一个符号数组：

```c
#define NSYMBOLS        1024            // Number of symbol table entries
extern_ struct symtable Gsym[NSYMBOLS]; // Global symbol table
static int Globs = 0;                   // Position of next free global symbol slot
```

`Globs` 实际上在 `sym.c` 文件中，该文件管理符号表。在这里我们有这些管理函数：

- `int findglob(char *s)`：确定符号 `s` 是否在全局符号表中。返回其槽位，如果未找到则返回 -1。
- `static int newglob(void)`：获取一个新的全局符号槽的位置，否则如果我们已经耗尽位置就会死亡。
- `int addglob(char *name)`：向符号表中添加一个全局符号。返回符号表中的槽位号。

代码相当直接，所以我不打算在讨论中给出代码。通过这些函数，我们可以找到符号并向符号表中添加新的符号。

## 扫描和新 token

如果查看示例输入文件，我们需要几个新 token：

- 'int'，对应 T_INT
- '='，对应 T_EQUALS
- 标识符名称，对应 T_IDENT

'=' 的扫描很容易添加到 `scan()`:

```c
  case '=':
    t->token = T_EQUALS; break;
```

我们可以在 `keyword()` 中添加 'int' 关键字：

```c
  case 'i':
    if (!strcmp(s, "int"))
      return (T_INT);
    break;
```

对于标识符，我们已经使用 `scanident()` 将单词存储到 `Text` 变量中。如果一个单词不是关键字，那么我们可以返回一个 `T_IDENT` 标记：

```c
   if (isalpha(c) || '_' == c) {
      // Read in a keyword or identifier
      scanident(c, Text, TEXTLEN);

      // If it's a recognised keyword, return that token
      if (tokentype = keyword(Text)) {
        t->token = tokentype;
        break;
      }
      // Not a recognised keyword, so it must be an identifier
      t->token = T_IDENT;
      break;
    }
```

## 新语法

我们已经准备好查看输入语言的语法变化了。和以前一样，我用 BNF 符号来定义它：

```c
 statements: statement
      |      statement statements
      ;

 statement: 'print' expression ';'
      |     'int'   identifier ';'
      |     identifier '=' expression ';'
      ;

 identifier: T_IDENT
      ;
```

标识符作为 `T_IDENT`  token返回，我们已经有了解析 print 语句的代码。但是，由于我们现在有三种类型的语句，因此有必要编写一个函数来处理每一种语句。`stmt.c` 中的顶层函数  `statements()` 现在看起来像：

```c
// Parse one or more statements
void statements(void) {

  while (1) {
    switch (Token.token) {
    case T_PRINT:
      print_statement();
      break;
    case T_INT:
      var_declaration();
      break;
    case T_IDENT:
      assignment_statement();
      break;
    case T_EOF:
      return;
    default:
      fatald("Syntax error, token", Token.token);
    }
  }
}
```

我已经将旧的 print 语句代码移到 `print_statement()` 中，你可以自己浏览。

## 变量声明

让我们看看变量声明。这是在一个新文件 `decl.c` 中，因为我们将来会有很多其他类型的声明。

```c
// Parse the declaration of a variable
void var_declaration(void) {

  // Ensure we have an 'int' token followed by an identifier
  // and a semicolon. Text now has the identifier's name.
  // Add it as a known identifier
  match(T_INT, "int");
  ident();
  addglob(Text);
  genglobsym(Text);
  semi();
}
```

`ident()` 和 `semi()` 函数是 `match()` 的包装器：

```c
void semi(void)  { match(T_SEMI, ";"); }
void ident(void) { match(T_IDENT, "identifier"); }
```

回到 `var_declaration()` ，一旦我们将标识符扫描到文本缓冲区中，我们就可以用 `addglob(Text)` 将它添加到全局符号表中。这里的代码允许一个变量被声明多次（目前）。

## 赋值语句

下面是 `stmt.c` 中 `assignment_statement()` 的代码：

```c
void assignment_statement(void) {
  struct ASTnode *left, *right, *tree;
  int id;

  // Ensure we have an identifier
  ident();

  // Check it's been defined then make a leaf node for it
  if ((id = findglob(Text)) == -1) {
    fatals("Undeclared variable", Text);
  }
  right = mkastleaf(A_LVIDENT, id);

  // Ensure we have an equals sign
  match(T_EQUALS, "=");

  // Parse the following expression
  left = binexpr(0);

  // Make an assignment AST tree
  tree = mkastnode(A_ASSIGN, left, right, 0);

  // Generate the assembly code for the assignment
  genAST(tree, -1);
  genfreeregs();

  // Match the following semicolon
  semi();
}
```

我们有几个新的 AST 节点类型。A_ASSIGN 获取左孩子中的表达式，并将其赋给右孩子。右孩子将是 A_LVIDENT 节点。

为什么我把这个节点叫做 A_LVIDENT？因为它表示一个左值标识符。那么什么是[左值](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue)呢？

左值是与特定位置相关联的值。在这里，它是内存中保存变量值的地址。当我们这样做时：

```c
   area = width * height;
```

我们将右边的结果（即右值）赋给左边的变量（即左值）。右值不依赖于特定的位置。这里，表达式结果可能在某个任意的寄存器中。

还要注意，虽然赋值语句的语法是：

```c
  identifier '=' expression ';'
```

我们将使表达式成为 A_ASSIGN 节点的左子树，并将 A_LVIDENT 细节保存在右子树中。为什么？因为我们需要在将表达式保存到变量中之前对其求值。

## AST 结构的变化

我们现在需要在 A_INTLIT AST 节点中存储一个整数字面量，或者存储 A_IDENT AST 节点的符号的详细信息。为此，我在 AST 结构中添加了一个 union (在 `defs.h` 中)：

```c
// Abstract Syntax Tree structure
struct ASTnode {
  int op;                       // "Operation" to be performed on this tree
  struct ASTnode *left;         // Left and right child trees
  struct ASTnode *right;
  union {
    int intvalue;               // For A_INTLIT, the integer value
    int id;                     // For A_IDENT, the symbol slot number
  } v;
};
```

## 生成赋值代码

现在让我们看看 `gen.c` 中对 `genAST()` 的更改：

```c
int genAST(struct ASTnode *n, int reg) {
  int leftreg, rightreg;

  // Get the left and right sub-tree values
  if (n->left)
    leftreg = genAST(n->left, -1);
  if (n->right)
    rightreg = genAST(n->right, leftreg);

  switch (n->op) {
  ...
    case A_INTLIT:
    return (cgloadint(n->v.intvalue));
  case A_IDENT:
    return (cgloadglob(Gsym[n->v.id].name));
  case A_LVIDENT:
    return (cgstorglob(reg, Gsym[n->v.id].name));
  case A_ASSIGN:
    // The work has already been done, return the result
    return (rightreg);
  default:
    fatald("Unknown AST operator", n->op);
  }

```

注意，我们首先计算左边的 AST 子树，然后得到一个保存左边子树值的寄存器号。现在我们把这个寄存器号传递给右边的子树。我们需要为 A_LVIDENT 节点这样做，以便 `cg.c` 中的 `cgstorglob() ` 函数知道哪个寄存器保存赋值表达式的右值结果。

因此，考虑这个 AST 树：

```
           A_ASSIGN
          /        \
     A_INTLIT   A_LVIDENT
        (3)        (5)
```

我们调用 `leftreg = genAST(n->left, -1);` 来进行 A_INTLIT 操作。这将 `return (cgloadint(n->v.intvalue));`，即加载一个值为 3 的寄存器并返回寄存器 id。

然后，我们调用 `righttreg = genAST(n->right, leftreg);` 来进行 A_LVIDENT 操作。这将 `return (cgstorglob(reg, Gsym[n->v.id].name));`，即将寄存器存储到名称在 `Gsym[5]` 中的变量中。

然后我们切换到 A_ASSIGN 的情况。嗯，我们所有的工作都已经完成了。右值仍然在一个寄存器中，所以让我们把它留在那里并返回它。稍后，我们将能够进行这样的表达：

```c
  a= b= c = 0;
```

其中赋值不仅仅是一个语句，还是一个表达式。

## 生成 x86-64 代码

你可能已经注意到，我把旧的 `cgload()` 函数的名字改为 `cgloadint()`。这是更具体的。现在我们有了一个函数来加载全局变量的值（在 `cg.c` 中）：

```c
int cgloadglob(char *identifier) {
  // Get a new register
  int r = alloc_register();

  // Print out the code to initialise it
  fprintf(Outfile, "\tmovq\t%s(\%%rip), %s\n", identifier, reglist[r]);
  return (r);
}
```

类似地，我们需要一个函数来将寄存器保存到一个变量中：

```c
// Store a register's value into a variable
int cgstorglob(int r, char *identifier) {
  fprintf(Outfile, "\tmovq\t%s, %s(\%%rip)\n", reglist[r], identifier);
  return (r);
}
```

我们还需要一个函数来创建一个新的全局整型变量：

```c
// Generate a global symbol
void cgglobsym(char *sym) {
  fprintf(Outfile, "\t.comm\t%s,8,8\n", sym);
}
```

当然，我们不能让解析器直接访问这些代码。相反，在 `gen.c` 的通用代码生成器中有一个函数充当接口：

```c
void genglobsym(char *s) { cgglobsym(s); }
```

## 表达式中的变量

现在我们可以赋值给变量了。但是我们如何将变量的值转化为表达式呢？我们已经有一个 `primary()`  函数来获取整型字面量。让我们修改它来加载一个变量的值：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;
  int id;

  switch (Token.token) {
  case T_INTLIT:
    // For an INTLIT token, make a leaf AST node for it.
    n = mkastleaf(A_INTLIT, Token.intvalue);
    break;

  case T_IDENT:
    // Check that this identifier exists
    id = findglob(Text);
    if (id == -1)
      fatals("Unknown variable", Text);

    // Make a leaf AST node for it
    n = mkastleaf(A_IDENT, id);
    break;

  default:
    fatald("Syntax error, token", Token.token);
  }

  // Scan in the next token and return the leaf node
  scan(&Token);
  return (n);
}
```

注意在 `T_IDENT` 情况下的语法检查，以确保在尝试使用变量之前已经声明了它。

还要注意，检索变量值的 AST 叶节点是 A_IDENT 节点。保存到变量中的叶节点是 A_LVIDENT 节点。这是右值和左值的区别。

## 尝试一下

我想这就是变量声明的全部内容了，所以让我们用 `input02` 文件来试试：

```
int fred;
int jim;
fred= 5;
jim= 12;
print fred + jim;
```

我们可以执行 `make test` ：

```
$ make test
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
               stmt.c sym.c tree.c
...
./comp1 input02
cc -o out out.s
./out
17
```

可以看到，我们计算了 `fred+jim`，即 5+12 或 17。以下是 `out.s` 中新的汇编代码：

```
        .comm   fred,8,8                # Declare fred
        .comm   jim,8,8                 # Declare jim
        ...
        movq    $5, %r8
        movq    %r8, fred(%rip)         # fred = 5
        movq    $12, %r8
        movq    %r8, jim(%rip)          # jim = 12
        movq    fred(%rip), %r8
        movq    jim(%rip), %r9
        addq    %r8, %r9                # fred + jim
```

## 其他修改

我可能还做了一些其他的改变。我唯一能记住的是在 `misc.c` 中创建了一些辅助函数。以便更容易报告致命错误：

```
// Print out fatal messages
void fatal(char *s) {
  fprintf(stderr, "%s on line %d\n", s, Line); exit(1);
}

void fatals(char *s1, char *s2) {
  fprintf(stderr, "%s:%s on line %d\n", s1, s2, Line); exit(1);
}

void fatald(char *s, int d) {
  fprintf(stderr, "%s:%d on line %d\n", s, d, Line); exit(1);
}

void fatalc(char *s, int c) {
  fprintf(stderr, "%s:%c on line %d\n", s, c, Line); exit(1);
}
```

## 总结与下一步

因此，这是一项艰巨的工作。我们必须编写符号表管理的初始部分。我们必须处理两种新的语句类型。我们必须添加一些新的 token 和一些新的 AST 节点类型。最后，我们必须添加一些代码来生成正确的 x86-64 程序集输出。

尝试编写几个输入文件示例，看看编译器是否正常工作，尤其是当它检测到语法错误和语义错误时（使用变量但没有声明）。

在编译器编写旅程的下一部分，我们将在语言中添加六个比较运算符。这将使我们能够从后面的部分开始研究控制结构。
