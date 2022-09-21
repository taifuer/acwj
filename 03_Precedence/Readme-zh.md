# Part 3：运算符优先级

在编译器编写过程的前一部分中，我们看到，解析器不一定强制执行语言的语义。它只执行语法的句法和结构规则。

我们最终得到的代码计算了错误的表达式值，如 `2*3+4*5`，因为代码创建了一个类似于：

```
     *
    / \
   2   +
      / \
     3   *
        / \
       4   5
```

而不是

```
          +
         / \
        /   \
       /     \
      *       *
     / \     / \
    2   3   4   5
```

为了解决这个问题，我们必须向解析器添加代码来执行运算符优先级。至少有两种方法可以做到这一点:

- 在语言的语法中显式地设置运算符优先级
- 使用运算符优先级表影响现有解析器

## 显式设置运算符优先级

下面是我们上节课的语法：

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

注意，这四个数学运算符之间没有区别。让我们调整一下语法，让它们有所不同：

```
expression: additive_expression
    ;

additive_expression:
      multiplicative_expression
    | additive_expression '+' multiplicative_expression
    | additive_expression '-' multiplicative_expression
    ;

multiplicative_expression:
      number
    | number '*' multiplicative_expression
    | number '/' multiplicative_expression
    ;

number:  T_INTLIT
         ;
```

现在我们有了两种表达式：加法表达式和乘法表达式。注意，语法现在强制数字只能作为乘法表达式的一部分。这就迫使 `*` 和 `/` 运算符与两边的数字绑定得更紧密，因此具有更高的优先级。

任何加法表达式实际上要么是一个乘法表达式，要么是一个加法（乘法）表达式后跟一个 `+` 或 `-` 运算符，然后是另一个乘法表达式。加法表达式现在比乘法表达式具有低得多的优先级。

## 在递归下降解析器中执行上述操作

如何使用上述版本的语法并在递归下降解析器中将其实现呢？我已经在文件 `expr2.c` 中完成了此操作，我将在下面介绍代码。

答案是使用 `multiplicative_expr()` 函数处理 `*` 和 `/` 运算符，使用 `additive_expr()` 函数处理优先级较低的 `+` 和 `-` 运算符。

这两个函数都要读入一些东西和一个运算符。然后，当后面的运算符具有相同的优先级时，每个函数将解析更多的输入，并用第一个运算符组合左右两半。

但是，`additive_expr()` 必须遵从优先级更高的 `multiplicative_expr()` 函数。下面是如何做到的代码。

## `additive_expr()`

```c
// Return an AST tree whose root is a '+' or '-' binary operator
struct ASTnode *additive_expr(void) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the left sub-tree at a higher precedence than us
  left = multiplicative_expr();

  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // Loop working on token at our level of precedence
  while (1) {
    // Fetch in the next integer literal
    scan(&Token);

    // Get the right sub-tree at a higher precedence than us
    right = multiplicative_expr();

    // Join the two sub-trees with our low-precedence operator
    left = mkastnode(arithop(tokentype), left, right, 0);

    // And get the next token at our precedence
    tokentype = Token.token;
    if (tokentype == T_EOF)
      break;
  }

  // Return whatever tree we have created
  return (left);
}
```

在一开始，我们立即调用 `multiplicative_expr()`，以防第一个运算符是高优先级的 `*` 或 `/`。该函数只会在遇到低优先级的 `+` 或 `-` 运算符时返回。

因此，当我们执行  `while`  循环时，我们知道我们有一个 `+` 或 `-` 运算符。循环直到输入中没有 token，即当我们碰到 `T_EOF` 标记时。

在循环内部，我们再次调用 `multiplicative_expr()`，以防后面运算符的优先级高于我们。同样，当不存在高优先级的运算符时，它将返回。

一旦我们有了左、右子树，我们就可以把它们和上次循环中得到的运算符结合起来。如此重复，所以如果我们有表达式 `2 + 4 + 6`，我们将得到 AST：

```
       +
      / \
     +   6
    / \
   2   4
```

但是如果 `multiplicative_expr()` 有自己的更高优先级的运算符，我们将组合子树和其中的多个节点。

## `multiplicative_expr()`

```
// Return an AST tree whose root is a '*' or '/' binary operator
struct ASTnode *multiplicative_expr(void) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // While the token is a '*' or '/'
  while ((tokentype == T_STAR) || (tokentype == T_SLASH)) {
    // Fetch in the next integer literal
    scan(&Token);
    right = primary();

    // Join that with the left integer literal
    left = mkastnode(arithop(tokentype), left, right, 0);

    // Update the details of the current token.
    // If no tokens left, return just the left node
    tokentype = Token.token;
    if (tokentype == T_EOF)
      break;
  }

  // Return whatever tree we have created
  return (left);
}
```

该代码类似于 `additive_expr()`，不同的是我们需要调用 `primary()` 来获取真正的整型字面量。

我们也只在运算符位于高优先级时进行循环。比如 `*` 和 `/` 运算符。一旦我们遇到一个低优先级运算符，我们就简单地返回到目前为止我们已经建立的子树。这又回到 `additive_expr()` 来处理低优先级运算符。

## 上述缺点

上面构造具有显式运算符优先级的递归下降解析器的方法可能效率很低，因为所有函数调用都需要达到正确的优先级。还必须有处理每一级运算符优先级的函数，所以我们最终会有很多行代码。

## 另一种选择：Pratt 解析

减少代码量的一种方法是使用 [Pratt 解析器](https://en.wikipedia.org/wiki/Pratt_parser)，它具有与每个标记相关联的优先级值表。

在这一点上，我强烈建议你阅读 Bob Nystrom 的[《Pratt Parsers: Expression Parsing Made Easy》](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)。Pratt 解析器仍然让我头疼，所以尽可能多地阅读，熟悉基本概念。

## `expr.c`：Pratt 解析

我已经在 `expr.c` 中实现了 Pratt 解析，这是对 `expr2.c` 的替代。

首先，我们需要一些代码来确定每个 token 的优先级：

```c
// Operator precedence for each token
static int OpPrec[] = { 0, 10, 10, 20, 20,    0 };
//                     EOF  +   -   *   /  INTLIT

// Check that we have a binary operator and
// return its precedence.
static int op_precedence(int tokentype) {
  int prec = OpPrec[tokentype];
  if (prec == 0) {
    fprintf(stderr, "syntax error on line %d, token %d\n", Line, tokentype);
    exit(1);
  }
  return (prec);
}
```

较大的数字（例如 20）意味着比较小的数字（例如 10）优先级更高。

现在，你可能会问：当有一个名为 `OpPrec[]`  的查找表时，为什么还要一个函数呢？答案是：为了发现语法错误。

假设输入是 `234 101 + 12`。我们可以扫描前两个 token。但是，如果我们只是使用 `OpPrec[]` 来获取第二个 token `101` 的优先级，那么就不会注意到它不是一个运算符。因此，`op_precedence()` 函数强制使用正确的语法。

现在，我们不再为每个优先级使用一个函数，而是使用一个使用运算符优先级表的表达式函数：

```c
// Return an AST tree whose root is a binary operator.
// Parameter ptp is the previous token's precedence.
struct ASTnode *binexpr(int ptp) {
  struct ASTnode *left, *right;
  int tokentype;

  // Get the integer literal on the left.
  // Fetch the next token at the same time.
  left = primary();

  // If no tokens left, return just the left node
  tokentype = Token.token;
  if (tokentype == T_EOF)
    return (left);

  // While the precedence of this token is
  // more than that of the previous token precedence
  while (op_precedence(tokentype) > ptp) {
    // Fetch in the next integer literal
    scan(&Token);

    // Recursively call binexpr() with the
    // precedence of our token to build a sub-tree
    right = binexpr(OpPrec[tokentype]);

    // Join that sub-tree with ours. Convert the token
    // into an AST operation at the same time.
    left = mkastnode(arithop(tokentype), left, right, 0);

    // Update the details of the current token.
    // If no tokens left, return just the left node
    tokentype = Token.token;
    if (tokentype == T_EOF)
      return (left);
  }

  // Return the tree we have when the precedence
  // is the same or lower
  return (left);
}
```

首先，请注意，这依然和前面的解析器函数一样是递归的。这一次，我们接收 在调用之前找到的 token 的优先级（参数 `ptp`）。`main()` 会用最低的优先级 0 调用我们，但我们将用更高的值调用自己。

你还应该注意到，代码与 `multiplicative_expr()` 函数非常相似：读入一个整型字面量，获取运算符的 token 类型，然后循环构建树。

区别在于循环条件和主体：

```c
multiplicative_expr():
  while ((tokentype == T_STAR) || (tokentype == T_SLASH)) {
    scan(&Token); right = primary();

    left = mkastnode(arithop(tokentype), left, right, 0);

    tokentype = Token.token;
    if (tokentype == T_EOF) return (left);
  }

binexpr():
  while (op_precedence(tokentype) > ptp) {
    scan(&Token); right = binexpr(OpPrec[tokentype]);

    left = mkastnode(arithop(tokentype), left, right, 0);

    tokentype = Token.token;
    if (tokentype == T_EOF) return (left);
  }
```

在 Pratt 解析器中，当下一个运算符的优先级高于当前 token 时，我们不只是使用 `primary()` 获取下一个整型字面量，而是使用 `binexpr(OpPrec[tokentype])` 调用自己来提高运算符的优先级。

一旦遇到了与我们当前的优先级相同或比当前优先级更低的 token，我们将简单地：

```c
  return (left);
```

这将是一个具有许多节点和运算符的子树，其优先级高于调用我们的运算符，也可能是一个与我们优先级相同的运算符的单个整型字面量。

现在我们有了一个函数来进行表达式解析。它使用一个小的辅助函数来强制运算符优先级，从而实现了我们语言的语义。

## 将两个解析器付诸行动

你可以创建两个程序，每个程序对应一个解析器：

```
$ make parser                                        # Pratt Parser
cc -o parser -g expr.c interp.c main.c scan.c tree.c

$ make parser2                                       # Precedence Climbing
cc -o parser2 -g expr2.c interp.c main.c scan.c tree.c
```

你还可以使用我们旅程的前一部分中的相同输入文件测试两个解析器：

```
$ make test
(./parser input01; \
 ./parser input02; \
 ./parser input03; \
 ./parser input04; \
 ./parser input05)
15                                       # input01 result
29                                       # input02 result
syntax error on line 1, token 5          # input03 result
Unrecognised character . on line 3       # input04 result
Unrecognised character a on line 1       # input05 result

$ make test2
(./parser2 input01; \
 ./parser2 input02; \
 ./parser2 input03; \
 ./parser2 input04; \
 ./parser2 input05)
15                                       # input01 result
29                                       # input02 result
syntax error on line 1, token 5          # input03 result
Unrecognised character . on line 3       # input04 result
Unrecognised character a on line 1       # input05 result
```

## 总结与展望

也许是时候退一步看看我们已经做了什么。我们现在有:

- 扫描器：识别并返回我们语言中的 token 

- 解析器：能够识别语法、报告语法错误并构建抽象语法树

- 优先级表：用于实现语言语义的解析器

- 解释器：它以深度优先的方式遍历抽象语法树并计算输入中的表达式的结果

我们还没有一个编译器。但是我们已经很接近制作我们的第一个编译器了!

在编译器编写过程的下一部分中，我们将替换解释器。取而代之的是，我们将编写一个转换器，为每个具有数学运算符的 AST 节点生成 x86-64 汇编代码。我们还将生成一些汇编前同步码和后同步码，以支持生成器输出的汇编代码。
