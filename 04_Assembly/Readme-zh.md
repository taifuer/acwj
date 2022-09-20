# Part 4：一个实际的编译器

是时候兑现我的承诺，真正编写一个编译器了。因此，在本部分中，我们将用生成 x86-64 汇编代码的代码替换程序中的解释器。

## 修改解释器

在我们这样做之前，有必要回顾下 `interp.c` 中的解释器代码：

```c
int interpretAST(struct ASTnode *n) {
  int leftval, rightval;

  if (n->left) leftval = interpretAST(n->left);
  if (n->right) rightval = interpretAST(n->right);

  switch (n->op) {
    case A_ADD:      return (leftval + rightval);
    case A_SUBTRACT: return (leftval - rightval);
    case A_MULTIPLY: return (leftval * rightval);
    case A_DIVIDE:   return (leftval / rightval);
    case A_INTLIT:   return (n->intvalue);

    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

`interpretAST()` 函数以深度优先的方式遍历给定的 AST 树。它先计算左子树，然后是右子树。最后，它使用当前树底部（base）的 `op` 值对这些子节点进行操作。

如果 `op` 值是四个数学运算符之一，则执行该数学运算符。如果 `op` 值表明节点只是一个整型字面量，则返回该字面量。

函数返回此树的最终值。而且，由于它是递归的，它会计算整个树的最终值，每次计算一个子树。

## 更改为汇编代码生成

我们将编写一个通用的汇编代码生成器。反过来，这将调用一组特定于 CPU 的代码生成函数。

以下是 `gen.c` 中的通用汇编代码生成器：

```c
// Given an AST, generate
// assembly code recursively
static int genAST(struct ASTnode *n) {
  int leftreg, rightreg;

  // Get the left and right sub-tree values
  if (n->left) leftreg = genAST(n->left);
  if (n->right) rightreg = genAST(n->right);

  switch (n->op) {
    case A_ADD:      return (cgadd(leftreg,rightreg));
    case A_SUBTRACT: return (cgsub(leftreg,rightreg));
    case A_MULTIPLY: return (cgmul(leftreg,rightreg));
    case A_DIVIDE:   return (cgdiv(leftreg,rightreg));
    case A_INTLIT:   return (cgload(n->intvalue));

    default:
      fprintf(stderr, "Unknown AST operator %d\n", n->op);
      exit(1);
  }
}
```

看起来很熟悉，是吧。我们正在进行相同的深度优先遍历。这次：

- A_INTLIT：将字面值加载到一个寄存器中
- 其他运算符：对保存左孩子和右孩子值的两个寄存器执行数学运算

## 调用 `genAST()`

`genAST()` 只计算给定给它的表达式的值。我们需要把最后的计算结果打印出来。我们还需要用一些前导代码（preamble，前导码）和一些尾随代码（postamble，后同步码）来包装生成的汇编代码。这可以用 `gen.c` 中的另一个函数来完成：

```c
void generatecode(struct ASTnode *n) {
  int reg;

  cgpreamble();
  reg = genAST(n);
  cgprintint(reg);      // Print the register with the result as an int
  cgpostamble();
}
```

## x86-64 代码生成器

现在我们需要看一些真实汇编代码的生成。目前，我的目标是 x86-64 CPU，因为这仍然是最常见的 Linux 平台之一。所以，打开 `cg.c`，让我们开始浏览。

### 分配寄存器

任何 CPU 的寄存器数量都有限。我们必须分配一个寄存器来保存整型字面值，以及对它们执行的任何计算。然而，一旦我们使用了一个值，我们通常可以丢弃该值，从而释放保存该值的寄存器。然后我们可以将该寄存器重新用于另一个值。

有三个函数处理寄存器分配：

- `freeall_registers()`：将所有寄存器设置为可用
- `alloc_register()`：分配一个空闲的寄存器
- `free_register()`：释放一个分配的寄存器

我不打算浏览代码，因为它很简单，但会进行一些错误检查。现在，如果寄存器用完，程序就会崩溃。稍后，我将处理空闲寄存器用完的情况。

该代码适用于通用寄存器：r0、r1、r2 和 r3。有一个包含实际寄存器名的字符串表：

```c
static char *reglist[4]= { "%r8", "%r9", "%r10", "%r11" };
```

这使得这些函数完全独立于 CPU 架构。

### 加载一个寄存器

这是在 `cgload()` 中完成的：一个寄存器被分配，然后一个 `movq` 指令将一个字面值加载到被分配的寄存器中。

```c
// Load an integer literal value into a register.
// Return the number of the register
int cgload(int value) {

  // Get a new register
  int r = alloc_register();

  // Print out the code to initialise it
  fprintf(Outfile, "\tmovq\t$%d, %s\n", value, reglist[r]);
  return(r);
}
```

### 两个寄存器相加

`cgadd()` 接受两个寄存器号，并生成将它们相加的代码。结果保存在两个寄存器中的一个，然后释放另一个以供将来使用：

```c
// Add two registers together and return
// the number of the register with the result
int cgadd(int r1, int r2) {
  fprintf(Outfile, "\taddq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1);
  return(r2);
}
```

注意加法是可交换的，所以我可以把 `r2` 加到 `r1` 上，而不是把 `r1` 加到 `r2` 上。返回具有最终值的寄存器的标识即可。

### 两个寄存器相乘

这与加法非常相似，而且运算是可交换的，因此可以返回任何寄存器：

```c
// Multiply two registers together and return
// the number of the register with the result
int cgmul(int r1, int r2) {
  fprintf(Outfile, "\timulq\t%s, %s\n", reglist[r1], reglist[r2]);
  free_register(r1);
  return(r2);
}
```

### 两个寄存器相减

减法是不可交换的：我们必须得到正确的顺序。从第一个寄存器中减去第二个寄存器，因此我们返回第一个寄存器并释放第二个：

```c
// Subtract the second register from the first and
// return the number of the register with the result
int cgsub(int r1, int r2) {
  fprintf(Outfile, "\tsubq\t%s, %s\n", reglist[r2], reglist[r1]);
  free_register(r2);
  return(r1);
}
```

### 两个寄存器相除

除法也是不可交换的，与减法一样。在 x86-64 上，它甚至更复杂。我们需要从 `r1` 中加载被除数 `%rax`。这需要使用 `cqo` 扩展到 8 个字节。然后，`idivq` 将用 `r2` 中的除数除以 `%rax`，留下 `%rax` 中的商，因此我们需要将其复制到 `r1` 或 `r2`。然后我们可以释放另一个寄存器。

```c
// Divide the first register by the second and
// return the number of the register with the result
int cgdiv(int r1, int r2) {
  fprintf(Outfile, "\tmovq\t%s,%%rax\n", reglist[r1]);
  fprintf(Outfile, "\tcqo\n");
  fprintf(Outfile, "\tidivq\t%s\n", reglist[r2]);
  fprintf(Outfile, "\tmovq\t%%rax,%s\n", reglist[r1]);
  free_register(r2);
  return(r1);
}
```

### 打印一个寄存器

没有将寄存器打印为十进制数的 x86-64 指令。为了解决这个问题，汇编前导代码包含一个名为 `printint()` 的函数，该函数接受一个寄存器参数并调用 `printf()` 以十进制形式打印结果。

我不打算给出 `cgpreamble()` 中的代码，但是它还包含 `main()` 的开始代码，这样我们就可以组装输出文件以得到一个完整的程序。`cgpostamble()` 的代码（这里也没有给出）只是调用 `exit(0)` 来结束程序。

然而，这里是 `cgprintint()`：

```c
void cgprintint(int r) {
  fprintf(Outfile, "\tmovq\t%s, %%rdi\n", reglist[r]);
  fprintf(Outfile, "\tcall\tprintint\n");
  free_register(r);
}
```

Linux x86-64 期望函数的第一个参数在 `%rdi` 寄存器中，因此我们在 `call printint` 之前将寄存器移到 `%rdi` 中。

## 进行第一次编译

以上就是 x86-64 代码生成器的全部内容。在 `main()` 中有一些额外的代码用于打开 `out.s` 作为输出文件。我还把解释器留在程序中，这样我们就可以确认我们的汇编对输入表达式的计算结果与解释器相同。

让我们制作编译器并在  `input01` 上运行它：

```
$ make
cc -o comp1 -g cg.c expr.c gen.c interp.c main.c scan.c tree.c

$ make test
./comp1 input01
15
cc -o out out.s
./out
15
```

对第一个 15 是解释器的输出。第二个 15 是汇编的输出。

## 检查汇编输出

那么，汇编输出到底是什么呢？下面是输入文件：

```
2 + 3 * 5 - 8 / 3
```

这是带有注释的  `out.s`：

```c
        .text                           # Preamble code
.LC0:
        .string "%d\n"                  # "%d\n" for printf()
printint:
        pushq   %rbp
        movq    %rsp, %rbp              # Set the frame pointer
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        movl    -4(%rbp), %eax          # Get the printint() argument
        movl    %eax, %esi
        leaq    .LC0(%rip), %rdi        # Get the pointer to "%d\n"
        movl    $0, %eax
        call    printf@PLT              # Call printf()
        nop
        leave                           # and return
        ret

        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp              # Set the frame pointer
                                        # End of preamble code

        movq    $2, %r8                 # %r8 = 2
        movq    $3, %r9                 # %r9 = 3
        movq    $5, %r10                # %r10 = 5
        imulq   %r9, %r10               # %r10 = 3 * 5 = 15
        addq    %r8, %r10               # %r10 = 2 + 15 = 17
                                        # %r8 and %r9 are now free again
        movq    $8, %r8                 # %r8 = 8
        movq    $3, %r9                 # %r9 = 3
        movq    %r8,%rax
        cqo                             # Load dividend %rax with 8
        idivq   %r9                     # Divide by 3
        movq    %rax,%r8                # Store quotient in %r8, i.e. 2
        subq    %r8, %r10               # %r10 = 17 - 2 = 15
        movq    %r10, %rdi              # Copy 15 into %rdi in preparation
        call    printint                # to call printint()

        movl    $0, %eax                # Postamble: call exit(0)
        popq    %rbp
        ret
```

我们现在有了一个合法的编译器：一个程序接受一种语言的输入，然后生成另一种语言的翻译。

我们仍然需要将输出组装成机器代码，并将其与支持库链接起来，但这是我们目前可以手动执行的工作。稍后，我们将编写一些代码来自动完成这项工作。

## 总结与展望

从解释器到通用代码生成器的改变是微不足道的，但是我们必须写一些代码来生成真正的汇编输出。为此，我们必须考虑如何分配寄存器：目前，我们有一个简单的解决方案。我们还必须处理一些 x86-64 的奇怪问题，比如 `idivq` 指令。

我还没有提到的是：为什么要为一个表达式生成 AST 呢？当然，当我们在 Pratt 解析器中遇到一个 `+` 标记时，我们可以调用 `cgadd()`，其他操作符也是如此。我要让你去思考这个问题，但我会在一两步后回来。

在编译器编写过程的下一部分，我们将向我们的语言添加一些语句，这样它就开始类似于一种合适的计算机语言。
