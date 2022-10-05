# Part 10：For 循环

在编写编译器的这一部分中，我将添加 FOR 循环。在讨论如何解决这个问题之前，我想先解释一下在实现方面需要解决的一个问题。

## FOR 循环语法

我假设你熟悉 FOR 循环的语法。一个例子是：

```c
  for (i=0; i < MAX; i++)
    printf("%d\n", i);
```

我将在我们的语言中使用 BNF 语法：

```
 for_statement: 'for' '(' preop_statement ';'
                          true_false_expression ';'
                          postop_statement ')' compound_statement  ;

 preop_statement:  statement  ;        (for now)
 postop_statement: statement  ;        (for now)
```

`preop_statement` 在循环开始之前运行。稍后，我们将不得不精确地限制这里可以执行的操作的类型（例如，没有 IF 语句）。然后计算 `true_false_expression`。如果为 `true`，循环执行 `compound_statement`。完成此操作后，将执行 `postop_statement`，代码返回来继续执行 `true_false_expression`。

## 问题

问题在于 `postop_statement` 是在 `compound_statemment` 之前解析的，但我们必须在  `compound_statemment` 的代码之后生成 `postop_sstatement` 的代码。

有几种方法可以解决这个问题。在我以前编写编译器时，我选择将 `compound_statement` 汇编代码放在一个临时缓冲区中，并在为 `postop_statement` 生成代码后放回缓冲区。在 SubC 编译器中，Nils 巧妙地利用标签，并跳转到标签来“线程化”代码的执行，以强制执行正确的序列。

但是我们在这里建立了一个 AST 树。让我们使用它以正确的顺序获得生成的汇编代码。

## 什么样的 AST 树？

你可能已经注意到 FOR 循环有四个结构组件：

1. `preop_statement`

2.  `true_false_expression`

3. `postop_statement`

4.  `compound_statement`

我真的不想再次更改 AST 节点结构以拥有四个子节点。但我们可以将 FOR 循环视为增强的 WHILE 循环：

```
   preop_statement;
   while ( true_false_expression ) {
     compound_statement;
     postop_statement;
   }
```

我们可以用现有的节点类型构建 AST 树来表示这种结构吗？是的：

```c
         A_GLUE
        /     \
   preop     A_WHILE
             /    \
        decision  A_GLUE
                  /    \
            compound  postop
```

从左到右自顶向下手动遍历这棵树，并使自己相信我们将以正确的顺序生成汇编代码。我们必须将 `compound_statement` 和 `postop_statement` 粘合在一起，以便当 WHILE 循环退出时，它将跳过`compound_statement` 和 `postop_statement`。

这也意味着我们需要一个新的 T_FOR token，但不需要一个新的 AST 节点类型。因此唯一的编译器更改将是扫描和解析。

## 标记和扫描

有一个新的关键字 'for' 和一个对应的标记 T_FOR。没有大的变化。

## 解析语句

我们确实需要对解析器进行结构上的修改。对于 For 语法，我只需要一个单语句解析器来解析 `preop_statement` 和 `postop_statement`。现在，我们有一个 `compound_statement()` 函数，它简单地循环，直到遇到一个右花括号 '}'。我们需要把它分开，这样 `compound_statement()` 调用 `single_statement()` 来获得一个语句。

但是还有一个问题。以 `assignment_statement()` 中赋值语句的现有解析为例。解析器必须在语句末尾找到一个分号。

这对于复合语句来说很好，但是对于 for 循环就不行了。我必须写下这样的内容：

```c
  for (i=1 ; i < 10 ; i= i + 1; )
```

因为每个赋值语句都必须以分号结尾。

我们需要的是单语句解析器不扫描分号，而是将其留给复合语句解析器。对于某些语句（例如赋值语句之间）我们使用分号扫描，而对于其他语句（例如连续的 IF 语句之间）则不使用分号扫描。

解释完所有这些之后，现在让我们看看新的单语句和复合语句解析代码:

```c
// Parse a single statement
// and return its AST
static struct ASTnode *single_statement(void) {
  switch (Token.token) {
    case T_PRINT:
      return (print_statement());
    case T_INT:
      var_declaration();
      return (NULL);		// No AST generated here
    case T_IDENT:
      return (assignment_statement());
    case T_IF:
      return (if_statement());
    case T_WHILE:
      return (while_statement());
    case T_FOR:
      return (for_statement());
    default:
      fatald("Syntax error, token", Token.token);
  }
}

// Parse a compound statement
// and return its AST
struct ASTnode *compound_statement(void) {
  struct ASTnode *left = NULL;
  struct ASTnode *tree;

  // Require a left curly bracket
  lbrace();

  while (1) {
    // Parse a single statement
    tree = single_statement();

    // Some statements must be followed by a semicolon
    if (tree != NULL &&
	(tree->op == A_PRINT || tree->op == A_ASSIGN))
      semi();

    // For each new tree, either save it in left
    // if left is empty, or glue the left and the
    // new tree together
    if (tree != NULL) {
      if (left == NULL)
	left = tree;
      else
	left = mkastnode(A_GLUE, left, NULL, tree, 0);
    }
    // When we hit a right curly bracket,
    // skip past it and return the AST
    if (Token.token == T_RBRACE) {
      rbrace();
      return (left);
    }
  }
}
```

我还删除了 `print_statement()` 和 `assignment_statement()` 中对 `semi()` 的调用。

## 解析 FOR 循环

考虑到上面 for 循环的 BNF 语法，这很简单。给定我们想要的 AST 树的形状，构建这个树的代码也很简单。代码如下：

```c
// Parse a FOR statement
// and return its AST
static struct ASTnode *for_statement(void) {
  struct ASTnode *condAST, *bodyAST;
  struct ASTnode *preopAST, *postopAST;
  struct ASTnode *tree;

  // Ensure we have 'for' '('
  match(T_FOR, "for");
  lparen();

  // Get the pre_op statement and the ';'
  preopAST= single_statement();
  semi();

  // Get the condition and the ';'
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    fatal("Bad comparison operator");
  semi();

  // Get the post_op statement and the ')'
  postopAST= single_statement();
  rparen();

  // Get the compound statement which is the body
  bodyAST = compound_statement();

  // For now, all four sub-trees have to be non-NULL.
  // Later on, we'll change the semantics for when some are missing

  // Glue the compound statement and the postop tree
  tree= mkastnode(A_GLUE, bodyAST, NULL, postopAST, 0);

  // Make a WHILE loop with the condition and this new body
  tree= mkastnode(A_WHILE, condAST, NULL, tree, 0);

  // And glue the preop tree to the A_WHILE tree
  return(mkastnode(A_GLUE, preopAST, NULL, tree, 0));
}
```

## 生成汇编代码

好了，我们所做的就是合成一个树，它有一个 WHILE 循环，一些子树粘在一起，所以编译器的生成端没有变化。

## 试一下

`tests/input07` 文件中包含这个程序：

```c
{
  int i;
  for (i= 1; i <= 10; i= i + 1) {
    print i;
  }
}
```

当我们执行 `make test` 时，会得到这样的输出：

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
    stmt.c sym.c tree.c
./comp1 tests/input07
cc -o out out.s
./out
1
2
3
4
5
6
7
8
9
10
```

下面是相关的汇编输出：

```assembly
	.comm	i,8,8
	movq	$1, %r8
	movq	%r8, i(%rip)		# i = 1
L1:
	movq	i(%rip), %r8
	movq	$10, %r9
	cmpq	%r9, %r8		# Is i < 10?
	jg	L2			# i >= 10, jump to L2
	movq	i(%rip), %r8
	movq	%r8, %rdi
	call	printint		# print i
	movq	i(%rip), %r8
	movq	$1, %r9
	addq	%r8, %r9		# i = i + 1
	movq	%r9, i(%rip)
	jmp	L1			# Jump to top of loop
L2:
```

## 总结与展望

在我们的语言中，现在有了一定数量的控制结构：IF 语句、WHILE 循环和 FOR 循环。问题是，接下来要解决什么？我们可以考虑的事情太多了: 

- 类型
- 局部与全局
- 函数
- 数组和指针
- 结构体和联合体
- auto，static 和 friend

我决定研究函数。因此，在编译器编写旅程的下一部分中，我们将开始向我们的语言添加函数的几个阶段中的第一个阶段。