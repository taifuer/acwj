# Part 15：指针，第 1 部分

在编译器编写过程的这一部分中，我想从为我们的语言添加指针开始。我特别要补充这些：

- 指针变量的声明
- 将地址赋值给指针
- 对指针进行解引用以获得它所指向的值

鉴于这是一项正在进行的工作，我确信我将实现一个目前可行的简单版本，但稍后我将不得不更改或扩展它以使其更通用。

## 新关键字和标记

这次没有新关键字，只有两个新标记：

- '&'，T_AMPER, 以及
- '&&'，T_LOGAND

我们还不需要 T_LOGAND，但我现在不妨将这段代码添加到 `scan()` 中：

```c
    case '&':
      if ((c = next()) == '&') {
        t->token = T_LOGAND;
      } else {
        putback(c);
        t->token = T_AMPER;
      }
      break;
```

## 类型的新代码

我在语言中添加了一些新的基本类型（在 `defs.h` 中）：

```c
// Primitive types
enum {
  P_NONE, P_VOID, P_CHAR, P_INT, P_LONG,
  P_VOIDPTR, P_CHARPTR, P_INTPTR, P_LONGPTR
};
```

我们将有新的一元前缀运算符：

- '&' 获取标识符的地址，并且
- '*' 来解引用指针并获取它所指向的值。

每个运算符产生的表达式类型与每个运算符所处理的类型不同。我们需要 `types.c` 中的两个函数来进行类型更改：

```c
// Given a primitive type, return
// the type which is a pointer to it
int pointer_to(int type) {
  int newtype;
  switch (type) {
    case P_VOID: newtype = P_VOIDPTR; break;
    case P_CHAR: newtype = P_CHARPTR; break;
    case P_INT:  newtype = P_INTPTR;  break;
    case P_LONG: newtype = P_LONGPTR; break;
    default:
      fatald("Unrecognised in pointer_to: type", type);
  }
  return (newtype);
}

// Given a primitive pointer type, return
// the type which it points to
int value_at(int type) {
  int newtype;
  switch (type) {
    case P_VOIDPTR: newtype = P_VOID; break;
    case P_CHARPTR: newtype = P_CHAR; break;
    case P_INTPTR:  newtype = P_INT;  break;
    case P_LONGPTR: newtype = P_LONG; break;
    default:
      fatald("Unrecognised in value_at: type", type);
  }
  return (newtype);
}
```

现在，我们要在哪里使用这些函数？

## 声明指针变量

我们希望能够声明标量变量和指针变量，例如：

```c
  char  a; char *b;
  int   d; int  *e;
```

`decl.c` 中已经有一个函数 `parse_type()` 将 type 关键字转换为类型。让我们将其扩展以扫描以下标记，如果下一个标记是 '*'，则更改类型：

```c
// Parse the current token and return
// a primitive type enum value. Also
// scan in the next token
int parse_type(void) {
  int type;
  switch (Token.token) {
    case T_VOID: type = P_VOID; break;
    case T_CHAR: type = P_CHAR; break;
    case T_INT:  type = P_INT;  break;
    case T_LONG: type = P_LONG; break;
    default:
      fatald("Illegal type, token", Token.token);
  }

  // Scan in one or more further '*' tokens 
  // and determine the correct pointer type
  while (1) {
    scan(&Token);
    if (Token.token != T_STAR) break;
    type = pointer_to(type);
  }

  // We leave with the next token already scanned
  return (type);
}
```

这将允许程序员尝试做：

```c
   char *****fred;
```

这将失败，因为 `pointer_to()` 还不能将 P_CHARPTR 转换为 P_CHARPTRPTR。但是 `parse_type()` 中的代码已经准备好去做了！

`var_declaration()` 中的代码现在可以很好地解析指针变量声明：

```c
// Parse the declaration of a variable
void var_declaration(void) {
  int id, type;

  // Get the type of the variable
  // which also scans in the identifier
  type = parse_type();
  ident();
  ...
}
```

### 前缀运算符 '*' 和 '&'

声明结束后，现在让我们看看解析表达式，其中 '*' 和 '&' 是出现在表达式前面的操作符。BNF 语法是这样的：

```c
 prefix_expression: primary
     | '*' prefix_expression
     | '&' prefix_expression
     ;
```

从技术上讲这允许：

```c
   x= ***y;
   a= &&&b;
```

为了防止这两个操作符不可能的使用，我们添加了一些语义检查。代码如下：

```c
// Parse a prefix expression and return 
// a sub-tree representing it.
struct ASTnode *prefix(void) {
  struct ASTnode *tree;
  switch (Token.token) {
    case T_AMPER:
      // Get the next token and parse it
      // recursively as a prefix expression
      scan(&Token);
      tree = prefix();

      // Ensure that it's an identifier
      if (tree->op != A_IDENT)
        fatal("& operator must be followed by an identifier");

      // Now change the operator to A_ADDR and the type to
      // a pointer to the original type
      tree->op = A_ADDR; tree->type = pointer_to(tree->type);
      break;
    case T_STAR:
      // Get the next token and parse it
      // recursively as a prefix expression
      scan(&Token); tree = prefix();

      // For now, ensure it's either another deref or an
      // identifier
      if (tree->op != A_IDENT && tree->op != A_DEREF)
        fatal("* operator must be followed by an identifier or *");

      // Prepend an A_DEREF operation to the tree
      tree = mkastunary(A_DEREF, value_at(tree->type), tree, 0);
      break;
    default:
      tree = primary();
  }
  return (tree);
}
```

我们仍然在做递归下降，但我们也加入了错误检查来防止输入错误。现在，`value_at()` 中的限制将防止一行中出现多个 '*' 操作符，但稍后当我们更改 `value_at()` 时，不必返回并更改 `prefix()`。

注意，当 `prefix()` 没有看到 '*' 或 '&' 操作符时，它也会调用 `primary()`。这允许我们在 `binexpr()` 中更改现有的代码：

```c
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int lefttype, righttype;
  int tokentype;

  // Get the tree on the left.
  // Fetch the next token at the same time.
  // Used to be a call to primary().
  left = prefix();
  ...
}
```

## 新的 AST 节点类型

在 `prefix()` 中，我引入了两种新的 AST 节点类型（在 `def.h` 中声明）：

- A_DEREF：对子节点中的指针进行解引用
- A_ADDR：获取该节点中标识符的地址

注意，A_ADDR 节点不是父节点。对于表达式 `&fred`，`prefix()` 中的代码将 "fred" 节点中的A_IDENT 替换为 A_ADDR 节点类型。

## 生成新的汇编代码

在我们的通用代码生成器 `gen.c` 中，`genAST()` 只有几行新代码：

```c
    case A_ADDR:
      return (cgaddress(n->v.id));
    case A_DEREF:
      return (cgderef(leftreg, n->left->type));
```

A_ADDR 节点生成代码来加载`n->v.id`标识符的地址放入寄存器。A_DEREF 节点接受 `lefreg` 中的指针地址及其关联类型，并返回具有该地址值的寄存器。

### x86-64 实现

通过查看其他编译器生成的汇编代码，我得出了以下汇编输出。这可能不正确！

```c
// Generate code to load the address of a global
// identifier into a variable. Return a new register
int cgaddress(int id) {
  int r = alloc_register();

  fprintf(Outfile, "\tleaq\t%s(%%rip), %s\n", Gsym[id].name, reglist[r]);
  return (r);
}

// Dereference a pointer to get the value it
// pointing at into the same register
int cgderef(int r, int type) {
  switch (type) {
    case P_CHARPTR:
      fprintf(Outfile, "\tmovzbq\t(%s), %s\n", reglist[r], reglist[r]);
      break;
    case P_INTPTR:
    case P_LONGPTR:
      fprintf(Outfile, "\tmovq\t(%s), %s\n", reglist[r], reglist[r]);
      break;
  }
  return (r);
}
```

`leaq` 指令加载指定标识符的地址。在段函数中，`(%r8)` 语法加载寄存器 %r8 指向的值。

## 测试新功能

这是我们新的测试文件 `tests/input15.c` 以及编译时的结果：

```c
int main() {
  char  a; char *b; char  c;
  int   d; int  *e; int   f;

  a= 18; printint(a);
  b= &a; c= *b; printint(c);

  d= 12; printint(d);
  e= &d; f= *e; printint(f);
  return(0);
}
```

我决定将测试文件改为以 `.c` 后缀结尾，因为它们实际上是 C 程序。我还更改了 `tests/mktests` 脚本，通过使用“真实”编译器编译测试文件来生成正确的结果。

## 总结与展望

我们实现了指针的开头。它们还不完全正确。例如，如果我编写以下代码：

```c
int main() {
  int x; int y;
  int *iptr;
  x= 10; y= 20;
  iptr= &x + 1;
  printint( *iptr);
}
```

它应该打印 20，因为 `&x + 1` 应该指向 `x` 后面的一个 `int` 数，即 `y`。这距离 `x` 有 8 个字节。然而，我们的编译器只是在 `x` 的地址上加 `1`，这是不正确的。我得想办法解决这个问题。

在编译器编写过程的下一部分中，我们将尝试修复这个问题。