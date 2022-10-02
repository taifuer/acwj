# Part 2：解析介绍



在编译器编写旅程的这一部分，我将介绍解析器（parser）的基础知识。正如我在第一部分中提到的，解析器的工作是识别输入的语法和结构元素，并确保它们符合语言的语法。

我们已经有几个可以扫描的元素，即我们的标记：

- 四个基本数学运算符：`*`、`/`、`+` 和 `-`
- 具有一个或多个数字 `0` .. `9` 的十进制整数

 现在，让我们为解析器将识别的语言定义语法。

## BNF：巴科斯-诺尔范式

如果你开始和计算机语言打交道，你会在某个时候碰到 [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form) (Backus-Naur Form) 的用法。我将在这里介绍足够的 BNF 语法来表达我们想要识别的语法。

我们需要一种语法来表达整数的数学表达式。下面是语法的 BNF 描述:

```
expression: number
          | expression '*' expression
          | expression '/' expression
          | expression '+' expression
          | expression '-' expression
          ;

number:  T_INTLIT
         ;
```

竖线分隔了语法中的选项，因此上面的内容是：

- 表达式可以只是一个数字，或者
- 表达式是由 '*' 标记分隔的两个表达式，或者
- 表达式是由 '/' 标记分隔的两个表达式，或者
- 表达式是由 '+' 标记分隔的两个表达式，或者
- 表达式是由 '-' 标记分隔的两个表达式
- 数字始终是 T_INTLIT 标记

显然，语法的 BNF 定义是*递归的*：表达式是通过引用其他表达式来定义的。但有一种方法可以“终止”递归：当一个表达式变成一个数字时，它总是一个 T_INTLIT 标记，因此不是递归的。

在 BNF 中，我们说“表达式”和“数字”是非终结符，因为它们是由语法规则产生的。然而，T_INTLIT 是终结符，因为它没有被任何规则定义。相反，它是语言中已经被识别的标记。类似地，四个数学运算符标记是终结符。

## 递归下降解析

鉴于我们语言的语法是递归的，尝试递归地解析它是有意义的。我们需要做的是读入一个标记，然后向前看下一个标记。根据下一个标记是什么，我们可以决定需要采用什么路径来解析输入。这可能需要我们递归地调用一个已经被调用的函数。

在我们的例子中，任何表达式中的第一个标记都是一个数字，后面可能跟着一个数学运算符。之后可能只有一个数字，也可能是一个全新表达式的开始。如何递归解析？

我们可以编写这样的伪代码：

```c
function expression() {
  Scan and check the first token is a number. Error if it's not
  Get the next token
  If we have reached the end of the input, return, i.e. base case

  Otherwise, call expression()
}
```

让我们在输入 `2 + 3 - 5 T_EOF ` 上运行这个函数，其中 `T_EOF` 是一个反映输入结束的标记（文件的末尾）。我会为每次对 `expression()` 的调用编号。

```c
expression0:
  Scan in the 2, it's a number
  Get next token, +, which isn't T_EOF
  Call expression()

    expression1:
      Scan in the 3, it's a number
      Get next token, -, which isn't T_EOF
      Call expression()

        expression2:
          Scan in the 5, it's a number
          Get next token, T_EOF, so return from expression2

      return from expression1
  return from expression0
```

是的，该函数能够递归地解析输入 `2 + 3 - 5 T_EOF`。

当然，我们没有对输入做任何处理，但这不是解析器的工作。解析器的工作是识别输入，并警告任何语法错误。其他操作将对输入进行语义分析，即理解并执行输入的含义。

> 稍后，你会发现这其实不是真的。将语法分析和语义分析结合起来通常是有意义的。

## 抽象语法树

为了进行语义分析，我们要么解释（interpret）已经识别的输入代码，要么将其转换为另一种格式，例如汇编代码。在本部分中，我们将为输入构建一个解释器（interpreter）。但要实现这一点，我们首先要将输入转换为[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)，也称为 AST (Abstract Syntax Trees)。

我强烈建议你阅读这篇关于 AST 的简短解释：

- [Leveling Up One’s Parsing Game With ASTs](https://medium.com/basecs/leveling-up-ones-parsing-game-with-asts-d7a6fc2400ff) by Vaidehi Joshi

它写得很好，对解释 AST 的目的和结构很有帮助。不用担心，你回来时我还在这。

对于我们将要构建的 AST，其中每个节点的结构在 `def.h` 中描述：

```c
// AST node types
enum {
  A_ADD, A_SUBTRACT, A_MULTIPLY, A_DIVIDE, A_INTLIT
};

// Abstract Syntax Tree structure
struct ASTnode {
  int op;                     // "Operation" to be performed on this tree
  struct ASTnode *left;       // Left and right child trees
  struct ASTnode *right;
  int intvalue;               // For A_INTLIT, the integer value
}; 
```

一些 AST 节点，如具有 `op` 值 `A_ADD` 和 `A_SUBTRACT` 的节点，有两个子 `AST`，分别由 `left` 和 `right` 指向。稍后，我们将 加上 或 减去 子树的值。

或者，具有 `op` 值为 `A_INTLIT` 的 `AST` 节点表示整数值。它没有子树，只有 `intvalue` 字段的一个值。

## 构建 AST 节点和树

 `tree.c` 中的代码有构建 AST 的函数。最通用的函数 `mkastnode()` 接受 AST 节点中所有四个字段的值。它分配节点，填充字段值并返回指向节点的指针：

```c
// Build and return a generic AST node
struct ASTnode *mkastnode(int op, struct ASTnode *left,
                          struct ASTnode *right, int intvalue) {
  struct ASTnode *n;

  // Malloc a new ASTnode
  n = (struct ASTnode *) malloc(sizeof(struct ASTnode));
  if (n == NULL) {
    fprintf(stderr, "Unable to malloc in mkastnode()\n");
    exit(1);
  }
  // Copy in the field values and return it
  n->op = op;
  n->left = left;
  n->right = right;
  n->intvalue = intvalue;
  return (n);
}
```

考虑到这一点，我们可以编写特定的函数来创建 AST 的叶节点（即没有子节点的节点），并创建具有单个子节点的 AST 节点：

```c
// Make an AST leaf node
struct ASTnode *mkastleaf(int op, int intvalue) {
  return (mkastnode(op, NULL, NULL, intvalue));
}

// Make a unary AST node: only one child
struct ASTnode *mkastunary(int op, struct ASTnode *left, int intvalue) {
  return (mkastnode(op, left, NULL, intvalue));
```

## AST 的目的

我们将使用 AST 来存储我们识别的每个表达式，以便稍后可以递归地遍历它以计算表达式的最终值。我们想要处理数学运算符的优先级。这里有一个例子。

考虑表达式 `2 * 3 + 4 * 5`。现在，乘法的优先级高于加法。因此，我们希望将乘法操作数绑定在一起，并在执行加法操作之前执行这些操作。

如果我们生成的 AST 树看起来像这样：

          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5

然后，在遍历树时，我们会先执行 `2*3`，然后执行 `4*5`。一旦得到这些结果，我们就可以将它们传递到树的根节点以执行加法。

## 一个简单的表达式解析器

现在，我们可以重用来自扫描器的标记值作为 AST 节点的操作值，但我喜欢将标记和 AST 节点的概念分离开来。因此，首先，我会有一个函数将标记值映射为 AST 节点的操作值。这和解析器的其余部分一起，都在 `expr.c` 中:

```c
// Convert a token into an AST operation.
int arithop(int tok) {
  switch (tok) {
    case T_PLUS:
      return (A_ADD);
    case T_MINUS:
      return (A_SUBTRACT);
    case T_STAR:
      return (A_MULTIPLY);
    case T_SLASH:
      return (A_DIVIDE);
    default:
      fprintf(stderr, "unknown token in arithop() on line %d\n", Line);
      exit(1);
  }
}
```

当不能将给定的标记转换为一种 AST 节点类型时，switch 语句中的默认语句将被触发。它将构成解析器中语法检查的一部分。

我们需要一个函数来检查下一个标记是否为整型字面值，并构建一个 AST 节点来保存字面值。这里是：

```c
// Parse a primary factor and return an
// AST node representing it.
static struct ASTnode *primary(void) {
  struct ASTnode *n;

  // For an INTLIT token, make a leaf AST node for it
  // and scan in the next token. Otherwise, a syntax error
  // for any other token type.
  switch (Token.token) {
    case T_INTLIT:
      n = mkastleaf(A_INTLIT, Token.intvalue);
      scan(&Token);
      return (n);
    default:
      fprintf(stderr, "syntax error on line %d\n", Line);
      exit(1);
  }
}
```

这里假设存在一个全局变量 `Token`，并且已经有了从输入中扫描进来的最新标记。在 `data.h` 中：

```c
extern_ struct token    Token;
```

以及在 `main()` 中:

```c
scan(&Token);                 // Get the first token from the input
n = binexpr();                // Parse the expression in the file
```

现在我们可以为解析器编写代码：

```c
// Return an AST tree whose root is a binary operator
struct ASTnode *binexpr(void) {
  struct ASTnode *n, *left, *right;
  int nodetype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  if (Token.token == T_EOF)
    return (left);

  // Convert the token into a node type
  nodetype = arithop(Token.token);

  // Get the next token in
  scan(&Token);

  // Recursively get the right-hand tree
  right = binexpr();

  // Now build a tree with both sub-trees
  n = mkastnode(nodetype, left, right, 0);
  return (n);
}
```

注意，在这个简单的解析器代码中，没有任何地方处理运算符优先级问题。就目前的情况而言，代码认为所有运算符都具有相同的优先级。如果你按照代码解析表达式  `2 * 3 + 4 * 5`，你会看到它构建了这个 AST:

```c
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

这肯定是不正确的。它会计算 `4*5` 得到 20，然后计算 `3+20` 得到 23，而不是计算 `2*3` 得到 6。

那我为什么要这么做？我想向你展示，编写一个简单的解析器很容易，但让它同时进行语义分析更难。

## 解释这棵树

现在我们有了（不正确的）AST 树，让我们编写一些代码来解释它。同样，我们将编写递归代码来遍历树。下面是伪代码：

```
interpretTree:
  First, interpret the left-hand sub-tree and get its value
  Then, interpret the right-hand sub-tree and get its value
  Perform the operation in the node at the root of our tree
  on the two sub-tree values, and return this value
```

回到正确的 AST 树：

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

调用结构如下所示：

```
interpretTree0(tree with +):
  Call interpretTree1(left tree with *):
     Call interpretTree2(tree with 2):
       No maths operation, just return 2
     Call interpretTree3(tree with 3):
       No maths operation, just return 3
     Perform 2 * 3, return 6

  Call interpretTree1(right tree with *):
     Call interpretTree2(tree with 4):
       No maths operation, just return 4
     Call interpretTree3(tree with 5):
       No maths operation, just return 5
     Perform 4 * 5, return 20

  Perform 6 + 20, return 26
```

## 解释树的代码

这在 `interp.c` 中，遵循上述伪代码：

```c
// Given an AST, interpret the
// operators in it and return
// a final value.
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  // Get the left and right sub-tree values
  if (n->left)
    leftval = interpretAST(n->left);
  if (n->right)
    rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:
      return (leftval + rightval);
    case A_SUBTRACT:
      return (leftval - rightval);
    case A_MULTIPLY:
      return (leftval * rightval);
    case A_DIVIDE:
      return (leftval / rightval);
    case A_INTLIT:
      return (n->intvalue);
    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

同样，当我们无法解释 AST 节点类型时，switch 语句中的默认语句将被触发。它将构成解析器中语义检查的一部分。

## 构建解析器

这里还有一些其他的代码，比如 `main()` 中对解释器的调用：

```
scan(&Token);                 // Get the first token from the input
n = binexpr();                // Parse the expression in the file
printf("%d\n", interpretAST(n));      // Calculate the final result
exit(0);
```

 你现在可以通过以下操作构建解析器：

```
$ make
cc -o parser -g expr.c interp.c main.c scan.c tree.c
```

我已经提供了几个输入文件供你测试解析器，当然你也可以创建自己的输入文件。记住，计算结果是不正确的，但解析器应该检测到输入错误，如连续数字、连续运算符和输入末尾缺少的数字。我还向解释器中添加了一些调试代码，这样你就可以看到 AST 树节点是按哪个顺序计算的：

```
$ cat input01
2 + 3 * 5 - 8 / 3

$ ./parser input01
int 2
int 3
int 5
int 8
int 3
8 / 3
5 - 2
3 * 3
2 + 9
11

$ cat input02
13 -6+  4*
5
       +
08 / 3

$ ./parser input02
int 13
int 6
int 4
int 5
int 8
int 3
8 / 3
5 + 2
4 * 7
6 + 28
13 - 34
-21

$ cat input03
12 34 + -56 * / - - 8 + * 2

$ ./parser input03
unknown token in arithop() on line 1

$ cat input04
23 +
18 -
45.6 * 2
/ 18

$ ./parser input04
Unrecognised character . on line 3

$ cat input05
23 * 456abcdefg

$ ./parser input05
Unrecognised character a on line 1
```

## 总结与展望

解析器识别语言的语法，并检查编译器的输入是否符合该语法。如果没有，解析器应该打印出一条错误消息。由于表达式语法是递归的，我们选择编写一个递归下降解析器来识别表达式。

现在解析器可以正常工作，如上面的输出所示，但是它不能正确地获取输入的语义。换句话说，它不能计算表达式的正确值。

在编译器编写过程的下一部分，我们将修改解析器，以便它也对表达式进行语义分析，以获得正确的数学结果。
