# Part 29：一点重构

我开始思考在编译器中实现 structs，unions 和 enums 的设计方面，然后我对如何改进符号表有了很好的想法，这导致我对编译器代码进行了一些重构。所以在这段旅程中没有新的功能，但我对编译器中的一些代码感到有点高兴。

如果你对我的 structs，unions 和 enums 的设计思想更感兴趣，请跳到下一部分。

## 重构符号表

当我开始编写我们的编译器时，我才通读完 [SubC](http://www.t3x.org/subc/) 编译器的代码并添加了我自己的注释。因此，我从这个代码库中借用了许多我最初的想法。其中一个是符号表的元素数组，一端是全局符号，另一端是局部符号。

我们已经看到，对于函数原型和参数，我们必须将函数的原型从全局端复制到局部端，这样函数就有了局部参数变量。我们还得担心符号表的一端会不会撞到另一端。

因此，在某种程度上，我们应该将符号表转换成多个单链表：至少一个用于全局符号，一个用于局部符号。当我们开始实现枚举时，我可能有第三个枚举值。

现在，我还没有在旅程的这一部分进行重构，因为变化看起来很大，所以我会等到真正需要的时候再做。但是我要做的另一个改变是这个。每个符号节点都有一个 `next` 指针来构成单链表，还有一个 `param` 指针。这将允许函数为它们的参数有一个单独的单链表，我们可以在搜索全局符号时跳过它。然后，当我们需要“复制”一个函数的原型作为它的参数列表时，我们可以简单地复制指向参数原型列表的指针。反正这个改变是为了以后。

## 类型，重温

我从 SubC 借用的另一个东西是类型的枚举（在`defs.h`中）：

```c
// Primitive types
enum {
  P_NONE, P_VOID, P_CHAR, P_INT, P_LONG,
  P_VOIDPTR, P_CHARPTR, P_INTPTR, P_LONGPTR
};
```

SubC 只允许一级间接寻址，因此上面列出了类型。我有一个想法，为什么不在原始类型值中编码间接级别？所以我改变了我们的代码，使得类型整数的底部四位是间接级别，而较高的位编码实际类型：

```c
// Primitive types. The bottom 4 bits is an integer
// value that represents the level of indirection,
// e.g. 0= no pointer, 1= pointer, 2= pointer pointer etc.
enum {
  P_NONE, P_VOID=16, P_CHAR=32, P_INT=48, P_LONG=64
};
```

我已经能够完全重构旧代码中的所有旧`P_XXXPTR`引用。让我们看看有什么变化。

首先，我们必须处理 `types.c`中的标量和指针类型。现在的代码实际上比以前更小：

```c
// Return true if a type is an int type
// of any size, false otherwise
int inttype(int type) {
  return ((type & 0xf) == 0);
}

// Return true if a type is of pointer type
int ptrtype(int type) {
  return ((type & 0xf) != 0);
}

// Given a primitive type, return
// the type which is a pointer to it
int pointer_to(int type) {
  if ((type & 0xf) == 0xf)
    fatald("Unrecognised in pointer_to: type", type);
  return (type + 1);
}

// Given a primitive pointer type, return
// the type which it points to
int value_at(int type) {
  if ((type & 0xf) == 0x0)
    fatald("Unrecognised in value_at: type", type);
  return (type - 1);
}
```

`modify_type()`没有任何变化。

在`express.c`中，当处理字面值字符串时，我使用了 `P_CHARPTR`，但现在我可以写：

```c
   n = mkastleaf(A_STRLIT, pointer_to(P_CHAR), id);
```

使用`P_XXXPTR`值的另一个重要区域是`cg.c`中依赖于硬件的代码中的代码。我们首先重写`cgprimsize()`来使用`ptrtype()`：

```c
// Dereference a pointer to get the value it
// pointing at into the same register
int cgderef(int r, int type) {

  // Get the type that we are pointing to
  int newtype = value_at(type);

  // Now get the size of this type
  int size = cgprimsize(newtype);

  switch (size) {
  case 1:
    fprintf(Outfile, "\tmovzbq\t(%s), %s\n", reglist[r], reglist[r]);
    break;
  case 2:
    fprintf(Outfile, "\tmovslq\t(%s), %s\n", reglist[r], reglist[r]);
    break;
  case 4:
  case 8:
    fprintf(Outfile, "\tmovq\t(%s), %s\n", reglist[r], reglist[r]);
    break;
  default:
    fatald("Can't cgderef on type:", type);
  }
  return (r);
}
```

快速阅读`cg.c`并查找对`cgprimsize()`的调用。

## 双指针的一个例子

现在我们已经有了16个间接级别，我编写了一个测试程序来确认它们是否工作，`tests/input55.c`：

```c
int printf(char *fmt);

int main(int argc, char **argv) {
  int i;
  char *argument;
  printf("Hello world\n");

  for (i=0; i < argc; i++) {
    argument= *argv; argv= argv + 1;
    printf("Argument %d is %s\n", i, argument);
  }
  return(0);
}
```

请注意，`argv++`还不起作用，`argv[i]`也不起作用。但我们可以解决这些缺失的特性，如上文所示。

## 对符号表结构的更改

虽然我没有将符号表重构为链表，但我确实调整了符号表结构本身，现在我意识到我可以使用 unions ，而不必为 unions 指定变量名：

```c
// Symbol table structure
struct symtable {
  char *name;                   // Name of a symbol
  int type;                     // Primitive type for the symbol
  int stype;                    // Structural type for the symbol
  int class;                    // Storage class for the symbol
  union {
    int size;                   // Number of elements in the symbol
    int endlabel;               // For functions, the end label
  };
  union {
    int nelems;                 // For functions, # of params
    int posn;                   // For locals, the negative offset
                                // from the stack base pointer
  };
};
```

我曾经为`nelems`定义了一个`#define`，但是上面是相同的结果，并且防止了`nelems`的全局定义污染名称空间。我还意识到`size`和`endlabel`可以在 struct 中占据相同的位置，并添加了该 union。对`addglob()`的参数做了一些修饰性的修改，但其他的就不多了。

## AST 结构的变化

类似地，我修改了 AST 节点结构，使 union 没有变量名：

```c
// Abstract Syntax Tree structure
struct ASTnode {
  int op;                       // "Operation" to be performed on this tree
  int type;                     // Type of any expression this tree generates
  int rvalue;                   // True if the node is an rvalue
  struct ASTnode *left;         // Left, middle and right child trees
  struct ASTnode *mid;
  struct ASTnode *right;
  union {                       // For A_INTLIT, the integer value
    int intvalue;               // For A_IDENT, the symbol slot number
    int id;                     // For A_FUNCTION, the symbol slot number
    int size;                   // For A_SCALE, the size to scale by
  };                            // For A_FUNCCALL, the symbol slot number
};
```

这意味着我可以写第二行而不是第一行：

```c
    return (cgloadglob(n->left->v.id, n->op));    // Old code
    return (cgloadglob(n->left->id,   n->op));    // New code
```

## 总结展望

这就是我们编译器编写旅程的这一部分。我可能在这里或那里做了一些更小的代码更改，但我想不出还有什么是主要的。

我将把符号表变成一个链表；这可能发生在我们实现枚举值的部分。

在我们的编译器编写旅程的下一部分，我将回到我想在这一部分讨论的内容：在我们的编译器中实现structs，unions 和 enums 的设计方面。