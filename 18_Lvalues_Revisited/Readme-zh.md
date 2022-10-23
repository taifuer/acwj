# Part 18：再探左值和右值

由于这是一项正在进行的工作，没有设计文档来指导我，所以偶尔我需要删除我已经编写的代码，并重写它以使它更通用，或者修复缺点。这部分旅程就是如此。

我们在第 15 部分中添加了对指针的初始支持，因此我们可以编写这样的代码行：

```
  int  x;
  int *y;
  int  z;
  x= 12; y= &x; z= *y;
```

这很好，但是我知道我们最终将不得不支持在赋值语句的左边使用指针，例如:

```c
  *y = 14;
```

要做到这一点，我们必须重新讨论[左值和右值](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue)的话题。回顾一下，左值是一个与特定位置相关联的值，而右值则不是。左值是持久的，因为我们可以在将来的指令中检索它们的值。另一方面，右值是短暂的：一旦用完，我们就可以丢弃它们。

### 右值和左值的例子

```c
   a            Scalar variable a
   b[0]         Element zero of array b
   *c           The location that pointer c points to
   (*d)[0]      Element zero of the array that d points to
```

正如我之前提到的，左值和右值来自赋值语句的两边：左值在左边，右值在右边。

## 扩展我们对左值的概念

现在，编译器几乎将所有东西都视为右值。对于变量，它从变量的位置检索值。我们对左值概念的唯一认可是将赋值左边的标识符标记为 A_LVIDENT。我们在 `genAST()` 中手工处理这个问题：

```c
    case A_IDENT:
      return (cgloadglob(n->v.id));
    case A_LVIDENT:
      return (cgstorglob(reg, n->v.id));
    case A_ASSIGN:
      // The work has already been done, return the result
      return (rightreg);
```

我们在诸如 `a=b;` 这样的语句中使用。但现在我们需要标记的不仅仅是赋值左侧的标识符作为左值。

在这个过程中使生成汇编代码变得容易也很重要。在写这一部分的时候，我尝试了将 "A_LVALUE" AST 节点作为树的父节点，告诉代码生成器输出代码的左值版本，而不是右值版本。但是这被证明为时已晚：子树已经被计算过了，它的右值代码已经生成了。

## 另一个 AST 节点变化

我不愿意继续向 AST 节点添加更多字段，但这就是我最终所做的。现在我们有一个字段来指示节点应该生成左值代码还是右值代码：

```c
// Abstract Syntax Tree structure
struct ASTnode {
  int op;                       // "Operation" to be performed on this tree
  int type;                     // Type of any expression this tree generates
  int rvalue;                   // True if the node is an rvalue
  ...
};
```

`rvalue` 字段仅保存一位信息；后面如果我需要存储其他布尔值，我将能够使用它作为一个位字段。

问：为什么我让字段指示节点的“右值”而不是“左值”？毕竟，我们的 AST 树中的大多数节点将保存右值而不是左值。当我在读 Nils Holm 关于 SubC 的书时，我读到了这样一句话：

> 因为间接寻址不能在以后被逆转，所以解析器假设每个部分表达式都是左值。

考虑处理语句 `b = a + 2` 的解析器。解析完 `b` 标识符后，我们还不能判断这是左值还是右值。直到我们点击了 `=` 标记，我们才能断定它是一个左值。

还有，C 语言允许赋值作为表达式，所以我们也可以写成 `b = c = a + 2`。同样，当我们解析`a`标识符时，在解析下一个标记之前，我们无法判断它是左值还是右值。

因此，我选择假设每个 AST 节点默认为左值。一旦我们可以明确地判断一个节点是否为右值，我们就可以设置右值字段来表明这一点。

## 赋值表达式

我还提到 C 语言允许赋值作为表达式。既然我们已经清楚了左值/右值的区别，我们可以将赋值的解析转换为语句，并将代码移入表达式解析器。我稍后会介绍这一点。

现在是时候看看为了实现这一切，对编译器代码库做了些什么。和往常一样，我们首先从标记和扫描器开始。

## 标记和扫描修改

我们这次没有新的标记或新的关键字。但是有一个影响标记代码的变化。`=`现在是一个两边都有表达式的二元运算符，所以我们需要将它与其他二元运算符集成在一起。

根据 [C 运算符的列表](https://en.cppreference.com/w/c/language/operator_precedence)，`=` 运算符的优先级比 `+` 或 `-` 低得多。我们需要重新排列我们的操作符列表和它们的优先级。在 `defs.h` 中：

```c
// Token types
enum {
  T_EOF,
  // Operators
  T_ASSIGN,
  T_PLUS, T_MINUS, ...
```

在 `expr.c` 中，我们需要更新保存二元运算符优先级的代码：

## 对解析器的修改

现在我们必须删除将赋值解析为语句，并将它们转换为表达式。我还擅自从语言中删除了 "print" 语句，我们现在可以称之为 `printint()`。因此，在`stmt.c`中，我已经删除了 `print_statement()` 和 `assignment_statement()`。

> 我还从语言中删除了 T_PRINT 和 'print' 关键字。现在左值和右值的概念不同了，我还删除了A_LVIDENT AST 节点类型。 

目前，`stmt.c`中`single_statement()`中的语句解析器假设接下来出现的是一个表达式，如果它不能识别第一个标记的话：

```c
static struct ASTnode *single_statement(void) {
  int type;

  switch (Token.token) {
    ...
    default:
    // For now, see if this is an expression.
    // This catches assignment statements.
    return (binexpr(0));
  }
}
```

这确实意味着 `2+3;` 将被视为合法声明。我们稍后会解决这个问题。在`compound_statement()`中，我们还确保表达式后跟分号：

```c
    // Some statements must be followed by a semicolon
    if (tree != NULL && (tree->op == A_ASSIGN ||
                         tree->op == A_RETURN || tree->op == A_FUNCCALL))
      semi();
```

## 表达式解析

您可能会认为，既然`=`被标记为二元表达式运算符，并且我们已经设置了它的优先级，那么我们就大功告成了。不是这样的！有两件事值得我们注意：

1. 我们需要在生成左边值的代码之前生成右边值的汇编代码。我们过去在语句解析器中这样做，现在我们必须在表达式解析器中这样做。
2. 赋值表达式是右结合的：运算符更紧密地绑定到右边的表达式，而不是左边的表达式。

我们以前没有接触过右结合律。让我们看一个例子。考虑 `2 + 3 + 4` 这个表达式。我们可以愉快地从左到右解析它，并构建 AST 树：

```
      +
     / \
    +   4
   / \
  2   3
```

对于表达式 `a= b= 3`，如果我们这样做，我们最终得到树：

```
      =
     / \
    =   3
   / \
  a   b
```

在将 3 赋值给左边的子树之前，我们不想做 `a= b`。相反，我们想要生成的是这棵树：

```
        =
       / \
      =   a
     / \
    3   b
```

我已经颠倒了叶节点，使其符合汇编输出顺序。我们首先将 3 存储在 `b` 中，然后将这次赋值的结果 3 存储在 `a` 中。

### 修改 Pratt 解析器

我们使用 Pratt 解析器来正确解析二元运算符的优先级。我进行了一项研究，以找出如何将右结合性添加到 Pratt 解析器中，并在 [Wikipedia](https://en.wikipedia.org/wiki/Operator-precedence_parser) 中找到了以下信息：

>    while lookahead is a binary operator whose precedence is greater than op's,
>    or a right-associative operator whose precedence is equal to op's

因此，对于右关联运算符，我们测试下一个运算符是否与我们要处理的运算符具有相同的优先级。这是对解析器逻辑的简单修改。我在 `expr.c` 中引入了一个新函数确定运算符是否为右关联：

```c
// Return true if a token is right-associative,
// false otherwise.
static int rightassoc(int tokentype) {
  if (tokentype == T_ASSIGN)
    return(1);
  return(0);
}
```

在`binexpr()`中，我们如前所述改变了 while 循环，并且我们还加入了一个 A_ASSIGN 特定的代码来交换周围的子树：

```c
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  struct ASTnode *ltemp, *rtemp;
  int ASTop;
  int tokentype;

  // Get the tree on the left.
  left = prefix();
  ...

  // While the precedence of this token is more than that of the
  // previous token precedence, or it's right associative and
  // equal to the previous token's precedence
  while ((op_precedence(tokentype) > ptp) ||
         (rightassoc(tokentype) && op_precedence(tokentype) == ptp)) {
    ...
    // Recursively call binexpr() with the
    // precedence of our token to build a sub-tree
    right = binexpr(OpPrec[tokentype]);

    ASTop = binastop(tokentype);
    if (ASTop == A_ASSIGN) {
      // Assignment
      // Make the right tree into an rvalue
      right->rvalue= 1;
      ...

      // Switch left and right around, so that the right expression's 
      // code will be generated before the left expression
      ltemp= left; left= right; right= ltemp;
    } else {
      // We are not doing an assignment, so both trees should be rvalues
      left->rvalue= 1;
      right->rvalue= 1;
    }
    ...
  }
  ...
}
```

还要注意将赋值表达式的右边显式标记为右值的代码。对于非赋值，表达式的两边都被标记为右值。

在`binexpr()`中还有几行代码，用于显式地将树设置为右值。当我们碰到一个叶节点时，这些就会被执行。例如`b= a;`中的`a`标识符；需要标记为右值，但是我们永远不会进入`while`循环体来这样做。

## 打印出树

这就是解析器的改变。我们现在有几个节点被标记为右值，有些节点根本没有被标记。此时，我意识到我在可视化生成的 AST 树时遇到了困难。我在 `tree.c` 中编写了一个名为`dumpAST()`的函数将每个 AST 树打印到标准输出。它并不复杂。编译器现在有一个`-T`命令行参数，它设置了一个内部标志`O_dumpAST`。以及`decl.c`中的`global_declarations()`代码。现在可以：

```c
       // Parse a function declaration and
       // generate the assembly code for it
       tree = function_declaration(type);
       if (O_dumpAST) {
         dumpAST(tree, NOLABEL, 0);
         fprintf(stdout, "\n\n");
       }
       genAST(tree, NOLABEL, 0);

```

树转储器代码按照遍历树的顺序打印出每个节点，所以输出不是树形的。但是，每个节点的缩进表示它在树中的深度。

让我们看一些赋值表达式的 AST 树的例子。我们从`a= b= 34;`开始：

```
      A_INTLIT 34
    A_WIDEN
    A_IDENT b
  A_ASSIGN
  A_IDENT a
A_ASSIGN
```

34足够小，可以作为一个字符大小的字面量，但它会变宽以匹配 `b` 的类型。`A_IDENT b`不表示“右值”，所以它是一个左值。值 34 存储在左值 `b` 中。然后，这个值被存储在左值 `a` 中。

现在我们来试试 `a = b+34;`：

```
    A_IDENT rval b
      A_INTLIT 34
    A_WIDEN
  A_ADD
  A_IDENT a
A_ASSIGN
```

现在可以看到 "rval b"，因此`b`的值被加载到寄存器中，而`b+34`表达式的结果存储在左值`a`中。

我们再来一次， `*x= *y`：

```
    A_IDENT y
  A_DEREF rval
    A_IDENT x
  A_DEREF
A_ASSIGN
```

标识符`y`被解引用，并且该右值被加载。然后将其存储在被解引用的左值`x`中。

## 将上述内容转换为代码

既然左值和右值节点已经清楚地标识出来了，我们可以将注意力转向如何将它们翻译成汇编代码。有许多像整数文字，加法等节点很明显是右值。`gen.c`中`genAST()`的代码需要担心的只是可能是左值的 AST 节点类型。以下是我对这些节点类型的了解：

```c
    case A_IDENT:
      // Load our value if we are an rvalue
      // or we are being dereferenced
      if (n->rvalue || parentASTop== A_DEREF)
        return (cgloadglob(n->v.id));
      else
        return (NOREG);

    case A_ASSIGN:
      // Are we assigning to an identifier or through a pointer?
      switch (n->right->op) {
        case A_IDENT: return (cgstorglob(leftreg, n->right->v.id));
        case A_DEREF: return (cgstorderef(leftreg, rightreg, n->right->type));
        default: fatald("Can't A_ASSIGN in genAST(), op", n->op);
      }

    case A_DEREF:
      // If we are an rvalue, dereference to get the value we point at
      // otherwise leave it for A_ASSIGN to store through the pointer
      if (n->rvalue)
        return (cgderef(leftreg, n->left->type));
      else
        return (leftreg);
```

## x86-64 代码生成器的更改

对`cg.c`的唯一改变是一个允许我们通过指针存储值的函数：

```c
// Store through a dereferenced pointer
int cgstorderef(int r1, int r2, int type) {
  switch (type) {
    case P_CHAR:
      fprintf(Outfile, "\tmovb\t%s, (%s)\n", breglist[r1], reglist[r2]);
      break;
    case P_INT:
      fprintf(Outfile, "\tmovq\t%s, (%s)\n", reglist[r1], reglist[r2]);
      break;
    case P_LONG:
      fprintf(Outfile, "\tmovq\t%s, (%s)\n", reglist[r1], reglist[r2]);
      break;
    default:
      fatald("Can't cgstoderef on type:", type);
  }
  return (r1);
}
```

## 总结与展望

对于旅程的这一部分，我想我选择了两三个不同的设计方向，尝试了它们，遇到了一个死胡同，并在我达到这里描述的解决方案之前退出了。我知道，在 SubC 中，Nils传递了一个“左值”结构，该结构保存了在任何时间点正在处理的 AST 树节点的“左值”性。但他的树只有一个表达式；这个编译器的 AST 树包含一个完整功能的节点。我确信，如果你查看其他三个编译器，你可能也会找到其他三个解决方案。

接下来我们可以做很多事情。有很多 C 运算符可以相对容易地添加到编译器中。我们有 A_SCALE，所以我们可以尝试结构。到目前为止，还没有局部变量，这在某些时候需要注意。而且，我们应该将函数一般化，使其具有多个参数和访问它们的能力。

在我们编译器编写旅程的下一部分，我想讨论数组。这将是解引用、左值和右值以及根据数组元素的大小缩放数组索引的组合。我们已经有了所有的语义组件，但是我们还需要添加标记、解析和实际的索引功能。这应该是一个有趣的话题，就像这个一样。

