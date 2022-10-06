# Part 11：函数，第 1 部分

我想开始将函数实现到我们的语言中，但我知道这需要很多步骤。在此过程中，我们必须处理的一些事情是：

- 数据类型：`char`，`int`，`long` 等
- 每个函数的返回类型
- 每个函数的参数数量 
- 函数局部变量与全局变量的比较

在我们旅程的这一部分，要做的事情太多了。所以我在这里要做的是，我们可以声明不同的函数。在我们生成的可执行文件中，只有 `main()` 函数会运行，但是我们将能够为多个函数生成代码。

希望很快，我们的编译器能够识别的语言将会成为 C 语言的一个子集，我们的输入将会被一个“真正的”C编译器所识别。但还不是时候。

## 简单的函数语法

这肯定是一个占位符，这样我们就可以解析一些看起来像函数的东西。一旦完成了这些，我们就可以添加其他重要的东西：类型、返回类型、参数等等。

因此，现在，我将在 BNF 中添加一个函数语法，如下所示：

```
function_declaration: 'void' identifier '(' ')' compound_statement   ;
```

所有函数都将被声明为 `void`，并且没有参数。我们也不会引入调用函数的能力，所以只有 `main()` 函数会执行。

我们需要一个新的关键字 `void` 和一个新的标记 T_VOID，这两个都很容易添加。

## 解析简单的函数语法

新函数的语法非常简单，我们可以编写一个漂亮的小函数来解析它（在 `dec.c` 中）：

```c
// Parse the declaration of a simplistic function
struct ASTnode *function_declaration(void) {
  struct ASTnode *tree;
  int nameslot;

  // Find the 'void', the identifier, and the '(' ')'.
  // For now, do nothing with them
  match(T_VOID, "void");
  ident();
  nameslot= addglob(Text);
  lparen();
  rparen();

  // Get the AST tree for the compound statement
  tree= compound_statement();

  // Return an A_FUNCTION node which has the function's nameslot
  // and the compound statement sub-tree
  return(mkastunary(A_FUNCTION, tree, nameslot));
}
```

这将进行语法检查和 AST 构建，但这里几乎没有语义错误检查。如果函数被重新声明了呢？好吧，我们还不会注意到。

## 对 main() 的修改

有了上面的函数，我们现在可以重写 main() 中的一些代码来逐个解析多个函数：

```c
scan(&Token);                 // Get the first token from the input
genpreamble();                // Output the preamble
while (1) {                   // Parse a function and
    tree = function_declaration();
    genAST(tree, NOREG, 0);     // generate the assembly code for it
    if (Token.token == T_EOF)   // Stop when we have reached EOF
        break;
}
```

注意，我已经删除了 `genpostamble()` 函数调用。这是因为它的输出在技术上是为 `main()` 生成的汇编的后文。我们现在需要一些代码生成函数来生成函数的开始（汇编代码）和结束（汇编代码）。

## 函数的通用代码生成

现在我们有了一个 A_FUNCTION AST 节点，我们最好在通用代码生成器 `gen.c` 中添加一些代码来处理它。从上面看，这是一个只有一个子节点的一元 AST 节点：

```c
// Return an A_FUNCTION node which has the function's nameslot
// and the compound statement sub-tree
return(mkastunary(A_FUNCTION, tree, nameslot));
```

子树包含复合语句，复合语句是函数的主体。在为复合语句生成代码之前，我们需要生成函数的开始（汇编代码）。下面是 `genAST()` 中实现这一点的代码：

```c
case A_FUNCTION:
// Generate the function's preamble before the code
cgfuncpreamble(Gsym[n->v.id].name);
genAST(n->left, NOREG, n->op);
cgfuncpostamble();
return (NOREG);
```

## x86-64 代码生成

现在，我们必须生成代码来为每个函数设置栈和帧指针，并在函数结束时撤销该操作，返回到函数的调用方。

我们已经在 `cgpreamble()` 和 `cgpostamble()` 中有了这段代码，但是 `cgpreamble()` 也有 `printint() `函数的汇编代码。因此，问题是将这些汇编代码片段分离为 `cg.c` 中的新函数：

```c
// Print out the assembly preamble
void cgpreamble() {
  freeall_registers();
  // Only prints out the code for printint()
}

// Print out a function preamble
void cgfuncpreamble(char *name) {
  fprintf(Outfile,
          "\t.text\n"
          "\t.globl\t%s\n"
          "\t.type\t%s, @function\n"
          "%s:\n" "\tpushq\t%%rbp\n"
          "\tmovq\t%%rsp, %%rbp\n", name, name, name);
}

// Print out a function postamble
void cgfuncpostamble() {
  fputs("\tmovl $0, %eax\n" "\tpopq     %rbp\n" "\tret\n", Outfile);
}
```

## 测试函数生成功能

我们有了一个新的测试程序 `tests/input08`，它开始看起来像一个 C 程序了（除了 `print` 语句）：

```c
void main()
{
  int i;
  for (i= 1; i <= 10; i= i + 1) {
    print i;
  }
}
```

要对此进行测试，请执行 `make test8`：

```c
cc -o comp1 -g cg.c decl.c expr.c gen.c main.c misc.c scan.c
    stmt.c sym.c tree.c
./comp1 tests/input08
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

我不打算查看汇编输出，因为它与上一部分中为 for 循环测试生成的代码相同。

然而，我已经将 `void main()` 添加到所有之前的测试输入文件中，因为该语言要求在复合语句代码之前声明一个函数。

测试程序 `tests/input09` 中声明了两个函数。编译器很乐意为每个函数生成有效的汇编代码，但目前我们还不能运行第二个函数的代码。

## 总结与展望

在给我们的语言增加函数方面，我们已经有了一个良好的开始。目前，它只是一个非常简单的函数声明。

在编译器编写旅程的下一部分，我们将开始向编译器添加类型。