# Part14：生成 ARM 汇编代码

在编译器编写过程的这一部分中，我将编译器移植到树莓派 4 上的 ARM CPU 上。

在本节开始之前，我应该说，虽然我非常了解 MIPS 汇编语言，但在开始这段旅程时，我只知道一点x86-32 汇编语言，对 x86-64 和 ARM 汇编语言一无所知。

一直以来，我所做的就是将示例 C 程序编译到使用各种 C 编译器的汇编程序中，看看它们生成的是哪种汇编语言。这就是我在这里为这个编译器编写 ARM 输出所做的。

## 主要的区别

首先，ARM是 RISC CPU，x86-64 是 CISC CPU。与 x86-64 相比，ARM 上的寻址模式更少。在生成ARM 汇编代码时，还会出现其他有趣的约束。所以我将从主要的不同点开始，把主要的相似点留到后面再说。

### ARM 寄存器

ARM 拥有比 x86-64 更多的寄存器堆。也就是说，我坚持分配四个寄存器：`r4`、`r5`、`r6` 和 `r7`。我们将看到 `r0` 和 `r3` 被用于下面的其他事情。

### 寻址全局变量

在 x86-64上 ，我们只需要用如下行声明一个全局变量:

```assembly
        .comm   i,4,4        # int variable
        .comm   j,1,1        # char variable
```

然后，我们可以轻松地加载和存储这些变量：

```c
        movb    %r8b, j(%rip)    # Store to j
        movl    %r8d, i(%rip)    # Store to i
        movzbl  i(%rip), %r8     # Load from i
        movzbq  j(%rip), %r8     # Load from j
```

使用 ARM，我们必须在程序后置代码中为所有全局变量手动分配空间:

```assembly
        .comm   i,4,4
        .comm   j,1,1
...
.L2:
        .word i
        .word j
```

要访问这些变量，我们需要用每个变量的地址加载一个寄存器，并从该地址加载第二个寄存器：

```assembly
        ldr     r3, .L2+0
        ldr     r4, [r3]        # Load i
        ldr     r3, .L2+4
        ldr     r4, [r3]        # Load j
```

存储变量是类似的：

```assembly
        mov     r4, #20
        ldr     r3, .L2+4
        strb    r4, [r3]        # i= 20
        mov     r4, #10
        ldr     r3, .L2+0
        str     r4, [r3]        # j= 10
```

现在在 `cgpostamble()` 中有这样的代码来生成 .words 表：

```c
  // Print out the global variables
  fprintf(Outfile, ".L2:\n");
  for (int i = 0; i < Globs; i++) {
    if (Gsym[i].stype == S_VARIABLE)
      fprintf(Outfile, "\t.word %s\n", Gsym[i].name);
  }
```

这也意味着我们需要为每个全局变量确定从 `.L2` 开始的偏移量。根据 KISS 原则，每次我想用变量的地址加载 `r3` 时，我都会手动计算偏移量。是的，我应该计算一次每个偏移量并将其存储在某个位置。

```c
// Determine the offset of a variable from the .L2
// label. Yes, this is inefficient code.
static void set_var_offset(int id) {
  int offset = 0;
  // Walk the symbol table up to id.
  // Find S_VARIABLEs and add on 4 until
  // we get to our variable

  for (int i = 0; i < id; i++) {
    if (Gsym[i].stype == S_VARIABLE)
      offset += 4;
  }
  // Load r3 with this offset
  fprintf(Outfile, "\tldr\tr3, .L2+%d\n", offset);
}
```

### 加载整型字面量

加载指令中的整型字面值的大小限制为 11 位，我认为这是一个有符号值。因此，我们不能将大整数字面值放入单个指令中。答案是将字面值像变量一样存储在内存中。所以我保留了一个以前使用的字面值列表。在后置代码中，我按照 `.L3` 标签输出它们。而且，与变量一样，我会遍历此列表以确定 `.L3` 标签中任何字面值的偏移量：

```c
// We have to store large integer literal values in memory.
// Keep a list of them which will be output in the postamble
#define MAXINTS 1024
int Intlist[MAXINTS];
static int Intslot = 0;

// Determine the offset of a large integer
// literal from the .L3 label. If the integer
// isn't in the list, add it.
static void set_int_offset(int val) {
  int offset = -1;

  // See if it is already there
  for (int i = 0; i < Intslot; i++) {
    if (Intlist[i] == val) {
      offset = 4 * i;
      break;
    }
  }

  // Not in the list, so add it
  if (offset == -1) {
    offset = 4 * Intslot;
    if (Intslot == MAXINTS)
      fatal("Out of int slots in set_int_offset()");
    Intlist[Intslot++] = val;
  }
  // Load r3 with this offset
  fprintf(Outfile, "\tldr\tr3, .L3+%d\n", offset);
}
```

### 函数前置代码

我将给你函数的前置代码，但我不完全确定每个指令是做什么的。这是针对 `int main(int x)` 的：

```assembly
  .text
  .globl        main
  .type         main, %function
  main:         push  {fp, lr}          # Save the frame and stack pointers
                add   fp, sp, #4        # Add sp+4 to the stack pointer
                sub   sp, sp, #8        # Lower the stack pointer by 8
                str   r0, [fp, #-8]     # Save the argument as a local var?
```

下面是返回单个值的函数后置代码：

```assembly
                sub   sp, fp, #4        # ???
                pop   {fp, pc}          # Pop the frame and stack pointers
```

### 返回 0 或 1 的比较

对于 x86-64，有一条指令可以根据比较结果将寄存器设置为 0 或 1，例如 `sete`，但是我们必须用 `movzbq` 将寄存器的其余部分归零。对于 ARM，我们运行两条单独的指令，如果我们想要的条件为真或假，则将寄存器设置为一个值，例如：

```assembly
                moveq r4, #1            # Set r4 to 1 if values were equal
                movne r4, #0            # Set r4 to 0 if values were not equal
```

## x86-64 和 ARM 汇编输出的比较

我想这就是所有主要的不同之处。因此，下面比较了 `cgXXX()` 操作、该操作的任何特定类型，以及执行该操作的示例 x86-64 和 ARM 指令序列。

| Operation(type)      | x86-64 Version        | ARM Version     |
| -------------------- | --------------------- | --------------- |
| cgloadint()          | movq $12, %r8         | mov r4, #13     |
| cgloadglob(char)     | movzbq foo(%rip), %r8 | ldr r3, .L2+#4  |
|                      |                       | ldr r4, [r3]    |
| cgloadglob(int)      | movzbl foo(%rip), %r8 | ldr r3, .L2+#4  |
|                      |                       | ldr r4, [r3]    |
| cgloadglob(long)     | movq foo(%rip), %r8   | ldr r3, .L2+#4  |
|                      |                       | ldr r4, [r3]    |
| int cgadd()          | addq %r8, %r9         | add r4, r4, r5  |
| int cgsub()          | subq %r8, %r9         | sub r4, r4, r5  |
| int cgmul()          | imulq %r8, %r9        | mul r4, r4, r5  |
| int cgdiv()          | movq %r8,%rax         | mov r0, r4      |
|                      | cqo                   | mov r1, r5      |
|                      | idivq %r8             | bl __aeabi_idiv |
|                      | movq %rax,%r8         | mov r4, r0      |
| cgprintint()         | movq %r8, %rdi        | mov r0, r4      |
|                      | call printint         | bl printint     |
|                      |                       | nop             |
| cgcall()             | movq %r8, %rdi        | mov r0, r4      |
|                      | call foo              | bl foo          |
|                      | movq %rax, %r8        | mov r4, r0      |
| cgstorglob(char)     | movb %r8, foo(%rip)   | ldr r3, .L2+#4  |
|                      |                       | strb r4, [r3]   |
| cgstorglob(int)      | movl %r8, foo(%rip)   | ldr r3, .L2+#4  |
|                      |                       | str r4, [r3]    |
| cgstorglob(long)     | movq %r8, foo(%rip)   | ldr r3, .L2+#4  |
|                      |                       | str r4, [r3]    |
| cgcompare_and_set()  | cmpq %r8, %r9         | cmp r4, r5      |
|                      | sete %r8              | moveq r4, #1    |
|                      | movzbq %r8, %r8       | movne r4, #1    |
| cgcompare_and_jump() | cmpq %r8, %r9         | cmp r4, r5      |
|                      | je L2                 | beq L2          |
| cgreturn(char)       | movzbl %r8, %eax      | mov r0, r4      |
|                      | jmp L2                | b L2            |
| cgreturn(int)        | movl %r8, %eax        | mov r0, r4      |
|                      | jmp L2                | b L2            |
| cgreturn(long)       | movq %r8, %rax        | mov r0, r4      |
|                      | jmp L2                | b L2            |

## 测试 ARM 代码生成器

如果你把编译器从这部分复制到树莓派 3 或 4，你应该能够：

```
$ make armtest
cc -o comp1arm -g -Wall cg_arm.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c types.c
cp comp1arm comp1
(cd tests; chmod +x runtests; ./runtests)
input01: OK
input02: OK
input03: OK
input04: OK
input05: OK
input06: OK
input07: OK
input08: OK
input09: OK
input10: OK
input11: OK
input12: OK
input13: OK
input14: OK

$ make armtest14
./comp1 tests/input14
cc -o out out.s lib/printint.c
./out
10
20
30
```

## 总结与展望

让代码生成器 `cg_arm.c` 的 ARM 版本正确编译所有测试输入确实花了我一些功夫。它基本上是直接的，我只是不熟悉架构和指令集。

将编译器移植到具有 3 或 4 个寄存器、2 个左右的数据大小和一个堆栈(和堆栈帧)的平台应该相对容易。接下来，我将尝试保持 `cg.c` 和 `cg_arm.c` 在功能上同步。

在编写编译器过程的下一部分中，我们将向语言添加 `char` 指针，以及 '*' 和 '&' 一元操作符。