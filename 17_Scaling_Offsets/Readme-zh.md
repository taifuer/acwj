# Part 17：更好的类型检查和指针偏移

前几部分，我引入了指针并实现了一些代码来检查类型兼容性。当时，我意识到，对于像这样的代码：

```c
  int   c;
  int  *e;

  e= &c + 1;
```

对由 `&c` 计算的指针加 1 需要转换为 `c` 的大小，以确保跳到内存中 `c` 之后的下一个 `int`。换句话说，我们必须对整数进行缩放。

我们需要对指针执行此操作，稍后我们需要对数组执行此操作。考虑代码：

```c
  int list[10];
  int x= list[3];
```

为此，我们需要找到`list[]`的基址，然后加上三倍于`int`的大小，以在索引位置`3`处找到元素。

当时，我在`types.c`中编写了一个名为`type_compatible()`的函数，以确定两种类型是否兼容，并指示是否需要“扩大”小整数类型，使其与大整数类型的大小相同。不过，这种扩大是在其他地方执行的。事实上，它最终在编译器的三个地方完成。

## `type_compatible()` 的替换

如果`type_compatible()`是这样指示的，我们将 A_WIDEN 一棵树以匹配更大的整数类型。现在我们需要 A_SCALE 一棵树，以便它的值按类型的大小进行缩放。并且我想重构复制的扩宽代码。

为此，我抛弃了`type_compatible()`并替换了它。这需要相当多的思考，我可能不得不再次调整或扩展它。让我们来看看设计。

现有的 `type_compatible()`：

- 接受两个类型值作为参数，加上一个可选的方向，
- 如果类型兼容，则返回 true，
- 如果任何一侧需要加宽，则在左侧或右侧返回 A_WIDEN ，
- 实际上并没有将 A_WIDEN 节点添加到树中，
- 如果类型不兼容，则返回 false，并且
- 不处理指针类型

现在让我们看看类型比较的用例：

- 当对两个表达式执行二元运算时，它们的类型是否兼容，我们是否需要加宽或缩放其中一个？
- 在执行 `print` 语句时，表达式是否为整数，是否需要加宽？
- 在执行赋值语句时，表达式是否需要加宽，它是否与左值类型匹配？
- 在执行 `return` 语句时，表达式是否需要加宽，它是否与函数的返回类型匹配？

在这些用例中，只有一个用例中有两个表达式。因此，我选择编写一个新函数，它接受一个 AST 树和我们希望它成为的类型。对于二元运算用例，我们将调用它两次，看看每次调用会发生什么。

## 引入 `modify_type()`

`types.c`中的`modify_type()`是`type_compatible()`的替换代码。该函数的 API 是：

```c
// Given an AST tree and a type which we want it to become,
// possibly modify the tree by widening or scaling so that
// it is compatible with this type. Return the original tree
// if no changes occurred, a modified tree, or NULL if the
// tree is not compatible with the given type.
// If this will be part of a binary operation, the AST op is not zero.
struct ASTnode *modify_type(struct ASTnode *tree, int rtype, int op);
```

问：为什么我们需要在这棵树和另一棵树上执行二元运算？答案是我们只能对指针进行加减运算。我们不能做别的事。

```c
  int x;
  int *ptr;

  x= *ptr;	   // OK
  x= *(ptr + 2);   // Two ints up from where ptr is pointing
  x= *(ptr * 4);   // Does not make sense
  x= *(ptr / 13);  // Does not make sense either
```

这是目前的代码。有很多具体的测试，目前我还没有找到一种方法来合理化所有可能的测试。另外，以后还需要进行扩展。

```c
struct ASTnode *modify_type(struct ASTnode *tree, int rtype, int op) {
  int ltype;
  int lsize, rsize;

  ltype = tree->type;

  // Compare scalar int types
  if (inttype(ltype) && inttype(rtype)) {

    // Both types same, nothing to do
    if (ltype == rtype) return (tree);

    // Get the sizes for each type
    lsize = genprimsize(ltype);
    rsize = genprimsize(rtype);

    // Tree's size is too big
    if (lsize > rsize) return (NULL);

    // Widen to the right
    if (rsize > lsize) return (mkastunary(A_WIDEN, rtype, tree, 0));
  }

  // For pointers on the left
  if (ptrtype(ltype)) {
    // OK is same type on right and not doing a binary op
    if (op == 0 && ltype == rtype) return (tree);
  }

  // We can scale only on A_ADD or A_SUBTRACT operation
  if (op == A_ADD || op == A_SUBTRACT) {

    // Left is int type, right is pointer type and the size
    // of the original type is >1: scale the left
    if (inttype(ltype) && ptrtype(rtype)) {
      rsize = genprimsize(value_at(rtype));
      if (rsize > 1) 
        return (mkastunary(A_SCALE, rtype, tree, rsize));
    }
  }

  // If we get here, the types are not compatible
  return (NULL);
}
```

添加 AST A_WIDEN 和 A_SCALE 操作的代码现在只在一个地方完成。A_WIDEN 操作将子节点的类型转换为父节点的类型。A_SCALE 操作将子节点的值乘以存储在新`struct ASTnode` union 字段中的大小（在 `defs.h` 中）：

```c
// Abstract Syntax Tree structure
struct ASTnode {
  ...
  union {
    int size;                   // For A_SCALE, the size to scale by
  } v;
};
```

## 使用新的 `modify_type()` API

使用这个新的 API，我们可以将 `stmt.c` 和 `expr.c` 中 A_WIDEN 的重复代码。但是，这个新函数只接受一棵树。当我们确实只有一棵树的时候，这很好。`stmt.c` 中现在有三个对 `modify_type()` 的调用，它们都很相似，所以下面是来自 `assignment_statement()` 的调用：

```c
  // Make the AST node for the assignment lvalue
  right = mkastleaf(A_LVIDENT, Gsym[id].type, id);

  ...
  // Parse the following expression
  left = binexpr(0);

  // Ensure the two types are compatible.
  left = modify_type(left, right->type, 0);
  if (left == NULL) fatal("Incompatible expression in assignment");
```

这比我们以前的代码要清晰得多。

### 在 `binexpr()` 中…

但是在 `expr.c` 中的 `binexpr()` 中，我们现在需要用加法、乘法等二元运算组合两棵 AST 树。在这里，我们尝试使用另一棵树的类型对每棵树进行 `modify_type()`。现在，一个可能会变宽：这也意味着另一个将失败并返回 NULL。因此，我们不能只看到 `modify_type()` 的一个结果是否为NULL：我们需要看到两个结果都为 NULL，以假定类型不兼容。以下是 `binexpr()` 中的新的比较代码：

```c
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  struct ASTnode *ltemp, *rtemp;
  int ASTop;
  int tokentype;

  ...
  // Get the tree on the left.
  // Fetch the next token at the same time.
  left = prefix();
  tokentype = Token.token;

  ...
  // Recursively call binexpr() with the
  // precedence of our token to build a sub-tree
  right = binexpr(OpPrec[tokentype]);

  // Ensure the two types are compatible by trying
  // to modify each tree to match the other's type.
  ASTop = arithop(tokentype);
  ltemp = modify_type(left, right->type, ASTop);
  rtemp = modify_type(right, left->type, ASTop);
  if (ltemp == NULL && rtemp == NULL)
    fatal("Incompatible types in binary expression");

  // Update any trees that were widened or scaled
  if (ltemp != NULL) left = ltemp;
  if (rtemp != NULL) right = rtemp;
```

## 执行缩放

我们已经将 A_SCALE 添加到 `defs.h` 中的 AST 节点运算列表中。现在我们需要实现它。

如前所述，A_SCALE 操作将子节点的值乘以存储在 `struct ASTnode` union 字段中的大小。对于我们所有的整数类型，这将是 2 的倍数。由于这个事实，我们可以将子值乘以一定位数的左移。

稍后，我们将有一个大小不是 2 的幂的结构。因此，当缩放因子合适时，我们可以进行移位优化，但我们还需要对更通用的缩放因子进行乘法运算。

以下是在 `gen.c` 的 `genAST()` 中执行此操作的新代码：

```c
    case A_SCALE:
      // Small optimisation: use shift if the
      // scale value is a known power of two
      switch (n->v.size) {
        case 2: return(cgshlconst(leftreg, 1));
        case 4: return(cgshlconst(leftreg, 2));
        case 8: return(cgshlconst(leftreg, 3));
        default:
          // Load a register with the size and
          // multiply the leftreg by this size
          rightreg= cgloadint(n->v.size, P_INT);
          return (cgmul(leftreg, rightreg));
```

## x86-64 代码中的左移

现在我们需要 `cgshlconst()` 函数来将寄存器值左移一个常数。当我们稍后添加 C '<<' 运算符时，我将编写一个更一般的左移函数。现在，我们可以使用带有整型字面值的 `salq` 指令：

```c
// Shift a register left by a constant
int cgshlconst(int r, int val) {
  fprintf(Outfile, "\tsalq\t$%d, %s\n", val, reglist[r]);
  return(r);
}
```

## 我们的测试程序不起作用

我对缩放功能的测试程序是 `tests/input16.c`：

```c
int   c;
int   d;
int  *e;
int   f;

int main() {
  c= 12; d=18; printint(c);
  e= &c + 1; f= *e; printint(f);
  return(0);
}
```

我希望当我们生成这些汇编指令时，`d` 会被汇编程序直接放在 `c` 后面：

```c
        .comm   c,1,1
        .comm   d,4,4
```

但当我编译汇编输出并进行检查时，它们并不相邻：

```c
$ cc -o out out.s lib/printint.c
$ nm -n out | grep 'B '
0000000000201018 B d
0000000000201020 B b
0000000000201028 B f
0000000000201030 B e
0000000000201038 B c
```

`d` 实际上在 `c` 之前！我必须找到一种方法来确保相邻，所以我查看了 SubC 在这里生成的代码，并更改了我们的编译器，现在生成如下代码：

```c
        .data
        .globl  c
c:      .long   0	# Four byte integer
        .globl  d
d:      .long   0
        .globl  e
e:      .quad   0	# Eight byte pointer
        .globl  f
f:      .long   0
```

现在，当我们运行`input16.c`测试时，`e = &c + 1; f = *e;` 获得从 `c` 后面一个的整数的地址，并将该整数的值存储在 `f` 中。

```c
  int   c;
  int   d;
  ...
  c= 12; d=18; printint(c);
  e= &c + 1; f= *e; printint(f);

```

我们将打印出这两个数字：

```
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c types.c
./comp1 tests/input16.c
cc -o out out.s lib/printint.c
./out
12
18
```

## 总结与展望

我对在类型之间转换的代码感到非常满意。在幕后，我编写了一些测试代码，为 `modify_type()` 提供了所有可能的类型值，以及一个二元运算和对应的零运算。我目测了一下输出，它似乎是我想要的。只有时间能证明一切。

在我们编译器编写旅程的下一部分，我不知道我会做什么！