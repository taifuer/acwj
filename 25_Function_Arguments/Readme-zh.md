# Part 25：函数调用与实参

在我们编译器编写旅程的这一部分，我将添加用任意数量的参数调用函数的能力，实参的值将被复制到函数的形参中，并作为局部变量出现。

我还没有这样做，因为在开始编码之前，还需要进行一些设计思考。让我们再一次回顾 Eli Bendersky 关于 [x86-64 上栈帧布局](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)的文章中的图片。

![](../22_Design_Locals/Figs/x64_frame_nonleaf.png)

通过寄存器 `%rdi` 到 `%r9`，最多可以传递六个函数的“按值调用”参数。对于六个以上的参数，剩余的参数被压入堆栈。

仔细查看堆栈上的参数值。尽管`h`是最后一个参数，但它首先被压入堆栈（向下增长），`g`参数在`h`参数之后被压入。

C 的一个不好的地方是没有定义表达式求值的顺序。[如下所述](https://en.cppreference.com/w/c/language/eval_order)：

> 任何 C 运算符的操作数的求值顺序，包括函数调用表达式中函数参数的求值顺序...未指定...。编译器会以任何顺序对它们求值...

这使得该语言潜在地不可移植：在一个平台上使用一个编译器的代码的行为可能在不同的平台上或在使用不同的编译器编译时具有不同的行为。

然而，对我们来说，缺少定义的求值顺序是*一件好事*，因为我们可以按照更容易编写编译器的顺序生成参数值。我在这里很轻率：这真的不是什么好事。

因为 x86-64 平台希望最后一个参数的值首先被推送到堆栈上，所以我需要编写代码来处理从最后一个到第一个的参数。我应该确保代码可以很容易地修改，以允许在另一个方向进行处理：也许可以编写 `genXXX()` 查询函数来告诉我们的代码处理参数的方向。我以后再写。

## 生成表达式的 AST

我们已经有了 A_GLUE AST 节点类型，所以编写一个函数来解析参数表达式并构建 AST 树应该很容易。对于一个函数调用 `function(expr1，expr2，expr3，expr4)`，我决定像这样构建树：

```
                 A_FUNCCALL
                  /
              A_GLUE
               /   \
           A_GLUE  expr4
            /   \
        A_GLUE  expr3
         /   \
     A_GLUE  expr2
     /    \
   NULL  expr1
```

每个表达式都在右边，前面的表达式在左边。我必须从右到左遍历表达式的子树，以确保在`expr3`之前处理`expr4`，以防前者在后者之前被 push 到 x86-64 堆栈上。

我们已经有了一个`funccall()`函数来解析一个总是有一个参数的简单函数调用。我将修改它来调用`expression_list()`函数以解析表达式列表并构建 A_GLUE 子树。它将通过在顶部 A_GLUE AST 节点中存储这个计数来返回表达式数量的计数。然后，在`funccall()`中，我们可以根据应该存储在全局符号表中的函数原型检查所有表达式的类型。

我认为这在设计方面已经足够了。现在让我们开始实现。

## 表达式解析更改

嗯，我在一个小时左右完成了代码，我感到惊喜。借用 Twitter 上流传的一句话：

> 数周的编程可以节省你数小时的计划时间。

相反，花一点时间在设计上总是有助于提高编码效率。让我们来看看变化。我们将从解析开始。

我们现在必须解析逗号分隔的表达式列表，并构建 A_GLUE AST 树，子表达式在右边，前面的表达式树在左边。下面是 `expr.c` 中的代码：

```c
// expression_list: <null>
//        | expression
//        | expression ',' expression_list
//        ;

// Parse a list of zero or more comma-separated expressions and
// return an AST composed of A_GLUE nodes with the left-hand child
// being the sub-tree of previous expressions (or NULL) and the right-hand
// child being the next expression. Each A_GLUE node will have size field
// set to the number of expressions in the tree at this point. If no
// expressions are parsed, NULL is returned
static struct ASTnode *expression_list(void) {
  struct ASTnode *tree = NULL;
  struct ASTnode *child = NULL;
  int exprcount = 0;

  // Loop until the final right parentheses
  while (Token.token != T_RPAREN) {

    // Parse the next expression and increment the expression count
    child = binexpr(0);
    exprcount++;

    // Build an A_GLUE AST node with the previous tree as the left child
    // and the new expression as the right child. Store the expression count.
    tree = mkastnode(A_GLUE, P_NONE, tree, NULL, child, exprcount);

    // Must have a ',' or ')' at this point
    switch (Token.token) {
      case T_COMMA:
        scan(&Token);
        break;
      case T_RPAREN:
        break;
      default:
        fatald("Unexpected token in expression list", Token.token);
    }
  }

  // Return the tree of expressions
  return (tree);
}
```

这比我预想的要容易编码得多。现在，我们需要将它与现有的函数调用解析器连接起来：

```c
// Parse a function call and return its AST
static struct ASTnode *funccall(void) {
  struct ASTnode *tree;
  int id;

  // Check that the identifier has been defined as a function,
  // then make a leaf node for it.
  if ((id = findsymbol(Text)) == -1 || Symtable[id].stype != S_FUNCTION) {
    fatals("Undeclared function", Text);
  }
  // Get the '('
  lparen();

  // Parse the argument expression list
  tree = expression_list();

  // XXX Check type of each argument against the function's prototype

  // Build the function call AST node. Store the
  // function's return type as this node's type.
  // Also record the function's symbol-id
  tree = mkastunary(A_FUNCCALL, Symtable[id].type, tree, id);

  // Get the ')'
  rparen();
  return (tree);
}
```

请注意`XXX`，它提醒我还有工作要做。解析器确实会检查函数是否已经被声明过，但是它还不会将参数类型与函数的原型进行比较。我很快会做那件事。

现在返回的 AST 树具有我在本文开头画出的形状。现在是时候遍历它并生成汇编代码了。

## 对通用代码生成器的更改

按照编译器的编写方式，AST 中的代码是与架构无关的，在`gen.c`中，实际的平台相关的后端是在`cg.c`中。所以我们从`gen.c`的变化开始。

遍历这个新的 AST 结构需要大量的代码，所以我现在有一个函数来处理函数调用。在`genAST()`中，我们现在有：

```c
  // n is the AST node being processed
  switch (n->op) {
    ...
    case A_FUNCCALL:
      return (gen_funccall(n));
  }
```

新 AST 结构的代码如下：

```c
// Generate the code to copy the arguments of a
// function call to its parameters, then call the
// function itself. Return the register that holds 
// the function's return value.
static int gen_funccall(struct ASTnode *n) {
  struct ASTnode *gluetree = n->left;
  int reg;
  int numargs=0;

  // If there is a list of arguments, walk this list
  // from the last argument (right-hand child) to the
  // first
  while (gluetree) {
    // Calculate the expression's value
    reg = genAST(gluetree->right, NOLABEL, gluetree->op);
    // Copy this into the n'th function parameter: size is 1, 2, 3, ...
    cgcopyarg(reg, gluetree->v.size);
    // Keep the first (highest) number of arguments
    if (numargs==0) numargs= gluetree->v.size;
    genfreeregs();
    gluetree = gluetree->left;
  }

  // Call the function, clean up the stack (based on numargs),
  // and return its result
  return (cgcall(n->v.id, numargs));
}
```

有几件事需要注意。我们通过在右子节点上调用`genAST()`来生成表达式代码。此外，我们将`numargs`设置为第一个`size`值，这是参数的数量（以一为基础，而不是以零为基础）。然后调用`cgcopyarg()`将这个值复制到函数的第 n 个形参中。一旦复制完成，就可以释放所有寄存器，为下一个表达式做准备，并遍历左子节点查找上一个表达式。

最后，我们运行`cgcall()`来生成对函数的实际调用。因为我们可能已经将参数值压入堆栈，所以我们向它提供了参数的总数，这样它就可以计算出有多少个要从堆栈中弹出。

这里没有特定于硬件的代码，但是，正如我在顶部提到的，我们正在从最后一个表达式到第一个表达式遍历表达式树。并不是所有的体系结构都希望这样，因此在计算顺序方面，还有使代码更加灵活的空间。

##  `cg.c`的更改

现在我们来看看生成实际`x86-64`汇编代码输出的函数。我们已经创建了一个新的`cgcopyarg()`，并修改了一个现有的`cgcall()`。

但首先，要提醒一下，我们有这些寄存器列表：

```c
#define FIRSTPARAMREG 9         // Position of first parameter register
static char *reglist[] =
  { "%r10", "%r11", "%r12", "%r13", "%r9", "%r8", "%rcx", "%rdx", "%rsi", "%rdi" };

static char *breglist[] =
  { "%r10b", "%r11b", "%r12b", "%r13b", "%r9b", "%r8b", "%cl", "%dl", "%sil", "%dil" };

static char *dreglist[] =
  { "%r10d", "%r11d", "%r12d", "%r13d", "%r9d", "%r8d", "%ecx", "%edx", "%esi", "%edi" };
```

当 FIRSTPARAMREG 设置为最后一个索引位置时：我们将沿着这个列表往回走。

另外，请记住，我们将得到的参数位置号是基于 1 的（即1,2,3,4, …）而不是基于 0 的（0,1,2,3, …），但上面的数组是基于 0 的。您将在下面的代码中看到一些 `+1` 或 `-1` 的调整。

这是 `cgcopyarg()`：

```c
// Given a register with an argument value,
// copy this argument into the argposn'th
// parameter in preparation for a future function
// call. Note that argposn is 1, 2, 3, 4, ..., never zero.
void cgcopyarg(int r, int argposn) {

  // If this is above the sixth argument, simply push the
  // register on the stack. We rely on being called with
  // successive arguments in the correct order for x86-64
  if (argposn > 6) {
    fprintf(Outfile, "\tpushq\t%s\n", reglist[r]);
  } else {
    // Otherwise, copy the value into one of the six registers
    // used to hold parameter values
    fprintf(Outfile, "\tmovq\t%s, %s\n", reglist[r],
            reglist[FIRSTPARAMREG - argposn + 1]);
  }
}
```

除了 `+1`，很好很简单。现在`cgcall()`的代码：

```c
// Call a function with the given symbol id
// Pop off any arguments pushed on the stack
// Return the register with the result
int cgcall(int id, int numargs) {
  // Get a new register
  int outr = alloc_register();
  // Call the function
  fprintf(Outfile, "\tcall\t%s\n", Symtable[id].name);
  // Remove any arguments pushed on the stack
  if (numargs>6) 
    fprintf(Outfile, "\taddq\t$%d, %%rsp\n", 8*(numargs-6));
  // and copy the return value into our register
  fprintf(Outfile, "\tmovq\t%%rax, %s\n", reglist[outr]);
  return (outr);
}
```

再一次，又好又简单。

## 测试

在编译器编写过程的最后一部分，我们有两个独立的测试程序`input27a.c`和`input27b.c`：我们必须用`gcc`编译其中一个。现在，我们可以将它们组合在一起，并用编译器编译它们。还有第二个测试程序`input28.c`，其中有更多的函数调用示例。一如既往地：

```c
$ make test
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c
    misc.c scan.c stmt.c sym.c tree.c types.c
(cd tests; chmod +x runtests; ./runtests)
  ...
input25.c: OK
input26.c: OK
input27.c: OK
input28.c: OK
```

## 总结与展望

现在，我觉得我们的编译器刚刚从一个“玩具”编译器变成了一个几乎有用的编译器：我们现在可以编写多函数程序并在函数之间调用。到那里需要几步，但我认为每一步都不是一个巨大的步骤。

显然，还有很长的路要走。我们需要添加结构体、联合体、外部标识符和预处理器。然后，我们必须使编译器更健壮，提供更好的错误检测，可能添加警告等。

在编译器编写之旅的下一部分，我想我将增加编写函数原型的能力。这将允许我们连接外部功能。我在考虑那些基于`int`和`char*`的原始 Unix 函数和系统调用，如 `open()`，`read()`，`write()`，`strcpy()` 等。用我们的编译器编译一些有用的程序会很好。