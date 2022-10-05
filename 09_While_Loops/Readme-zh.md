# Part 9：While 循环

在这段旅程中，我们将在语言中添加 WHILE 循环。从某种意义上来说，WHILE 循环非常类似于没有 'else' 子句的 IF 语句，只是我们总是跳回到循环的顶部。

所以，这个：

```c
  while (condition is true) {
    statements;
  }
```

应该被转换成：

```assembly
Lstart: evaluate condition
	jump to Lend if condition false
	statements
	jump to Lstart
Lend:
```

这意味着我们可以借用与 IF 语句一起使用的扫描、解析和代码生成结构，并进行一些小的更改来处理 WHILE 语句。

让我们看看我们是如何做到这一点的。

## 新的标记

对于新的 'while' 关键字，我们需要一个新的标记 T_WHILE。对 `defs.h` 和 `scan.c` 的更改非常明显，所以我在这里省略它们。

## 解析 while 语法

WHILE 循环的 BNF 语法为：

```
// while_statement: 'while' '(' true_false_expression ')' compound_statement  ;
```

我们需要 `stmt.c` 中的一个函数来解析它。请注意，与 IF 语句的解析相比，这很简单：

```c
// Parse a WHILE statement
// and return its AST
struct ASTnode *while_statement(void) {
  struct ASTnode *condAST, *bodyAST;

  // Ensure we have 'while' '('
  match(T_WHILE, "while");
  lparen();

  // Parse the following expression
  // and the ')' following. Ensure
  // the tree's operation is a comparison.
  condAST = binexpr(0);
  if (condAST->op < A_EQ || condAST->op > A_GE)
    fatal("Bad comparison operator");
  rparen();

  // Get the AST for the compound statement
  bodyAST = compound_statement();

  // Build and return the AST for this statement
  return (mkastnode(A_WHILE, condAST, NULL, bodyAST, 0));
}
```

我们需要一个新的 AST 节点类型 A_WHILE，它已被添加到 `def.h` 中。这个节点有一个用于计算条件的左子树，还有一个用于复合语句（WHILE 循环的主体）的右子树。

## 通用代码生成

我们需要创建一个开始和结束标签，计算条件并插入适当的跳转以退出循环并返回到循环的顶部。同样，这比生成 IF 语句的代码简单得多。在 `gen.c` 中：

```c
// Generate the code for a WHILE statement
// and an optional ELSE clause
static int genWHILE(struct ASTnode *n) {
  int Lstart, Lend;

  // Generate the start and end labels
  // and output the start label
  Lstart = label();
  Lend = label();
  cglabel(Lstart);

  // Generate the condition code followed
  // by a jump to the end label.
  // We cheat by sending the Lfalse label as a register.
  genAST(n->left, Lend, n->op);
  genfreeregs();

  // Generate the compound statement for the body
  genAST(n->right, NOREG, n->op);
  genfreeregs();

  // Finally output the jump back to the condition,
  // and the end label
  cgjump(Lstart);
  cglabel(Lend);
  return (NOREG);
}
```

我必须做的一件事是意识到比较运算符的父 AST 节点现在可以是 A_WHILE，因此在 `genAST()`中，比较运算符的代码如下所示：

```c
    case A_EQ:
    case A_NE:
    case A_LT:
    case A_GT:
    case A_LE:
    case A_GE:
      // If the parent AST node is an A_IF or A_WHILE, generate 
      // a compare followed by a jump. Otherwise, compare registers 
      // and set one to 1 or 0 based on the comparison.
      if (parentASTop == A_IF || parentASTop == A_WHILE)
        return (cgcompare_and_jump(n->op, leftreg, rightreg, reg));
      else
        return (cgcompare_and_set(n->op, leftreg, rightreg));
```

总之，这就是我们实现 WHILE 循环所需要的全部！

## 测试

我已经将所有输入文件移到 `test/` 目录中。如果现在进行 `make test`，它将进入此目录，编译每个输入，并将输出与已知的正确输出进行比较：

```bash
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c stmt.c
      sym.c tree.c
(cd tests; chmod +x runtests; ./runtests)
input01: OK
input02: OK
input03: OK
input04: OK
input05: OK
input06: OK
```

你也可以执行 `make test6`。这将编译 `tests/input06` 文件：

```
{ int i;
  i=1;
  while (i <= 10) {
    print i;
    i= i + 1;
  }
}
```

这将打印出 1 到 10 之间的数字：

```
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
      stmt.c sym.c tree.c
./comp1 tests/input06
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

下面是编译的汇编输出：

```assembly
	.comm	i,8,8
	movq	$1, %r8
	movq	%r8, i(%rip)		# i= 1
L1:
	movq	i(%rip), %r8
	movq	$10, %r9
	cmpq	%r9, %r8		# Is i <= 10?
	jg	L2			# Greater than, jump to L2
	movq	i(%rip), %r8
	movq	%r8, %rdi		# Print out i
	call	printint
	movq	i(%rip), %r8
	movq	$1, %r9
	addq	%r8, %r9		# Add 1 to i
	movq	%r9, i(%rip)
	jmp	L1			# and loop back
L2:
```

## 总结与展望

一旦我们已经完成了 IF 语句，WHILE 循环很容易添加，因为它们有很多相似之处。

我想我们现在也有了[图灵完备](https://en.wikipedia.org/wiki/Turing_completeness)语言：

- 无限的存储空间，也就是无限的变量
- 基于存储的值做出决策的能力，即 IF 语句
- 改变方向的能力，即 WHILE 循环

所以我们可以停止了，我们的任务完成了！不，当然不是。我们仍在努力让编译器可以编译自己。

在编译器编写过程的下一部分中，我们将向语言中添加 FOR 循环。