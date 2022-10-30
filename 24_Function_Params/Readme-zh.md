# Part 24：函数形参

我刚刚实现了将函数形参从寄存器中复制到函数的堆栈中，但是还没有实现带参数的函数调用。

这里是 Eli Bendersky 关于 [x86-64 上栈帧布局](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/)的文章中的图片。

![](../22_Design_Locals/Figs/x64_frame_nonleaf.png)

通过寄存器“%rdi”到“%r9”，最多可向函数传递六个“按值调用”参数。对于六个以上的参数，剩余的参数被压入堆栈。

调用该函数时，它将前一个堆栈基址指针（saved RBP） push 到堆栈上，将堆栈基址指针（RBP）向下移动到与堆栈指针（RSP）相同的位置，然后将堆栈指针移动到最低的（lowest）局部变量（at minimum）。

为什么是“at minimum”？嗯，我们还必须把堆栈指针降低到 16 的倍数，这样在我们调用另一个函数之前，堆栈基址指针才能正确对齐。

压入堆栈的参数将保留在那里，与堆栈基址指针的偏移量为正。我们将把所有通过寄存器传递的参数复制到堆栈上，并在堆栈上为我们的局部变量设置位置。这些将具有从堆栈基址指针的负偏移量。

这是目标，但我们必须先完成几件事。

## 新的标记和扫描

首先，ANSI C 中的函数声明是一个用逗号分隔的类型和变量名列表。

```c
int function(int x, char y, long z) { ... }
```

因此，我们需要一个新的标记 T_COMMA，并修改词法扫描器以读入它。我将让你阅读`scan.c`中对`scan()`的更改。

## 一个新的存储类

在编写编译器旅程的上一部分中，我描述了为支持全局变量和局部变量而对符号表进行的更改。我们在表的一端存储全局变量，在另一端存储局部变量。现在，我要引入函数形参。

我在`def.h`中添加了一个新的存储类定义：

```c
// Storage classes
enum {
        C_GLOBAL = 1,           // Globally visible symbol
        C_LOCAL,                // Locally visible symbol
        C_PARAM                 // Locally visible function parameter
};
```

它们将出现在符号表中的何处？实际上，相同的参数将出现在表的全局和局部末尾。

在全局符号列表中，我们将首先使用 C_GLOBAL、S_FUNCTION 条目定义函数的符号。然后，我们将使用标记为 C_PARAM 的连续条目定义所有形参。这是函数的原型。这意味着，当我们稍后调用函数时，我们可以将实参列表与形参列表进行比较，并确保它们匹配。

同时，相同的参数列表存储在本地符号列表中，标记为 C_PARAM 而不是 C_LOCAL。这使我们能够区分别人发送给我们的变量和我们自己声明的变量。

## 解析器的更改

在这部分旅程中，我只处理函数声明。为此，我们需要修改解析器。一旦我们解析了函数的类型、名称和开头的 '('，我们就可以查找任何参数。每个参数都是按照正常的变量声明语法声明的，但参数声明不是以分号结尾，而是以逗号分隔。
 `decl.c` 中的旧`var_declaration()`函数用于在变量声明末尾扫描`T_SEMI`令牌。现在，它已被移到`var_declaration()`的前一个调用方。
我们现在有了一个新函数`param_declaration()`，它的任务是读取左括号后面的（零个或多个）参数列表：

```c
// param_declaration: <null>
//           | variable_declaration
//           | variable_declaration ',' param_declaration
//
// Parse the parameters in parentheses after the function name.
// Add them as symbols to the symbol table and return the number
// of parameters.
static int param_declaration(void) {
  int type;
  int paramcnt=0;

  // Loop until the final right parentheses
  while (Token.token != T_RPAREN) {
    // Get the type and identifier
    // and add it to the symbol table
    type = parse_type();
    ident();
    var_declaration(type, 1, 1);
    paramcnt++;

    // Must have a ',' or ')' at this point
    switch (Token.token) {
      case T_COMMA: scan(&Token); break;
      case T_RPAREN: break;
      default:
        fatald("Unexpected token in parameter list", Token.token);
    }
  }

  // Return the count of parameters
  return(paramcnt);
}
```

`var_declaration()`的两个'1'参数表明这是一个局部变量，也是一个参数声明。在`var_declaration()`中，我们现在这样做：

```c
    // Add this as a known scalar
    // and generate its space in assembly
    if (islocal) {
      if (addlocl(Text, type, S_VARIABLE, isparam, 1)==-1)
       fatals("Duplicate local variable declaration", Text);
    } else {
      addglob(Text, type, S_VARIABLE, 0, 1);
    }
```

该代码过去允许重复的局部变量声明，但这现在将导致堆栈增长超过所需，因此我将任何重复声明视为致命错误。

## 符号表的更改

早些时候，我说过一个形参将同时放在符号表的全局和局部端，但是上面的代码只显示了对`addlocl()`的调用。那到底是怎么回事？

我修改了`addlocal()`，也添加了一个参数到全局端：

```c
int addlocl(char *name, int type, int stype, int isparam, int size) {
  int localslot, globalslot;
  ...
  localslot = newlocl();
  if (isparam) {
    updatesym(localslot, name, type, stype, C_PARAM, 0, size, 0);
    globalslot = newglob();
    updatesym(globalslot, name, type, stype, C_PARAM, 0, size, 0);
  } else {
    updatesym(localslot, name, type, stype, C_LOCAL, 0, size, 0);
  }
```

我们不仅在参数的符号表中获得了一个局部槽，而且还为它获得了全局槽。两者都标记为 C_PARAM，而不是 C_LOCAL。

鉴于全局端现在包含非 C_LOCAL 的符号，我们需要修改代码以搜索全局符号：

```c
// Determine if the symbol s is in the global symbol table.
// Return its slot position or -1 if not found.
// Skip C_PARAM entries
int findglob(char *s) {
  int i;

  for (i = 0; i < Globs; i++) {
    if (Symtable[i].class == C_PARAM) continue;
    if (*s == *Symtable[i].name && !strcmp(s, Symtable[i].name))
      return (i);
  }
  return (-1);
}
```

## x86-64 代码生成器的更改

这就是解析函数形参并在符号表中记录它们的存在的全部内容。现在我们需要生成一个合适的函数前导代码，它将寄存器内参数复制到堆栈上的位置，并设置新的堆栈基址指针和堆栈指针。

在上一部分写了`cgresetlocals()`之后，我意识到我可以在调用`cgfuncpredirect()`时重置堆栈偏移量，所以我删除了这个函数。另外，计算新局部变量偏移量的代码只需要在`cg.c`中可见，所以我将其重命名为：

```c
// Position of next local variable relative to stack base pointer.
// We store the offset as positive to make aligning the stack pointer easier
static int localOffset;
static int stackOffset;

// Create the position of a new local variable.
static int newlocaloffset(int type) {
  // Decrement the offset by a minimum of 4 bytes
  // and allocate on the stack
  localOffset += (cgprimsize(type) > 4) ? cgprimsize(type) : 4;
  return (-localOffset);
}
```

我也从计算负偏移转换到计算正偏移，因为这使得数学（在我的头脑中）更容易。我仍然返回一个负的偏移量，如返回值所示。

我们有六个新寄存器将保存参数值，所以我们最好在某个地方命名它们。我扩展了寄存器名称列表，如下所示：

```c
#define NUMFREEREGS 4
#define FIRSTPARAMREG 9         // Position of first parameter register
static int freereg[NUMFREEREGS];
static char *reglist[] =
  { "%r10", "%r11", "%r12", "%r13", "%r9", "%r8", "%rcx", "%rdx", "%rsi",
"%rdi" };
static char *breglist[] =
  { "%r10b", "%r11b", "%r12b", "%r13b", "%r9b", "%r8b", "%cl", "%dl", "%sil",
"%dil" };
static char *dreglist[] =
  { "%r10d", "%r11d", "%r12d", "%r13d", "%r9d", "%r8d", "%ecx", "%edx",
"%esi", "%edi" };
```

FIRSTPARAMREG 实际上是每个列表中的最后一个条目位置。我们将从这一端开始，反向工作。

现在我们将注意力转向将为我们完成所有工作的函数`cgfuncpredirect()`。让我们分阶段来看代码：

```c
// Print out a function preamble
void cgfuncpreamble(int id) {
  char *name = Symtable[id].name;
  int i;
  int paramOffset = 16;         // Any pushed params start at this stack offset
  int paramReg = FIRSTPARAMREG; // Index to the first param register in above reg lists

  // Output in the text segment, reset local offset
  cgtextseg();
  localOffset= 0;

  // Output the function start, save the %rsp and %rsp
  fprintf(Outfile,
          "\t.globl\t%s\n"
          "\t.type\t%s, @function\n"
          "%s:\n" "\tpushq\t%%rbp\n"
          "\tmovq\t%%rsp, %%rbp\n", name, name, name);
```

首先，声明函数，保存旧的基指针，并将其移动到当前堆栈指针所在的位置。我们还知道，任何栈上的参数都会比新的基址指针高 16，并且我们知道哪个是包含第一个参数的寄存器。

```c
  // Copy any in-register parameters to the stack
  // Stop after no more than six parameter registers
  for (i = NSYMBOLS - 1; i > Locls; i--) {
    if (Symtable[i].class != C_PARAM)
      break;
    if (i < NSYMBOLS - 6)
      break;
    Symtable[i].posn = newlocaloffset(Symtable[i].type);
    cgstorlocal(paramReg--, i);
  }
```

这最多循环六次，但是一旦我们遇到不是 C_PARAM 的东西，也就是 C_LOCAL，就会离开循环。调用`newlocaloffset()`从堆栈上的基指针生成一个偏移量，并将寄存器参数复制到堆栈上的这个位置。

```c
  // For the remainder, if they are a parameter then they are
  // already on the stack. If only a local, make a stack position.
  for (; i > Locls; i--) {
    if (Symtable[i].class == C_PARAM) {
      Symtable[i].posn = paramOffset;
      paramOffset += 8;
    } else {
      Symtable[i].posn = newlocaloffset(Symtable[i].type);
    }
  }
```

对于每个剩余的局部变量：如果它是一个 C_PARAM，那么它已经在堆栈上，所以只需记录它在符号表中的现有位置。如果是 C_LOCAL，在栈上创建一个新的位置并记录下来。我们现在已经用我们需要的所有局部变量建立了新的栈帧。剩下的就是将堆栈指针对齐 16 的倍数：

```c
  // Align the stack pointer to be a multiple of 16
  // less than its previous value
  stackOffset = (localOffset + 15) & ~15;
  fprintf(Outfile, "\taddq\t$%d,%%rsp\n", -stackOffset);
}
```

`stackOffset`是在整个`cg.c`中可见的静态变量。我们需要记住这个值，因为在函数的后同步时，我们需要将堆栈值增加我们降低它的量，并恢复旧的堆栈基址指针：

```c
// Print out a function postamble
void cgfuncpostamble(int id) {
  cglabel(Symtable[id].endlabel);
  fprintf(Outfile, "\taddq\t$%d,%%rsp\n", stackOffset);
  fputs("\tpopq %rbp\n" "\tret\n", Outfile);
}
```

## 测试

通过对编译器的这些修改，我们可以声明一个带有许多参数的函数以及我们需要的任何局部变量。但是编译器还没有生成在寄存器中传递参数的代码。

因此，为了测试我们的编译器的这一变化，我们编写了一些带参数的函数，并用我们的编译器编译它们（`input27a.c`）：

```c
int param8(int a, int b, int c, int d, int e, int f, int g, int h) {
  printint(a); printint(b); printint(c); printint(d);
  printint(e); printint(f); printint(g); printint(h);
  return(0);
}

int param5(int a, int b, int c, int d, int e) {
  printint(a); printint(b); printint(c); printint(d); printint(e);
  return(0);
}

int param2(int a, int b) {
  int c; int d; int e;
  c= 3; d= 4; e= 5;
  printint(a); printint(b); printint(c); printint(d); printint(e);
  return(0);
}

int param0() {
  int a; int b; int c; int d; int e;
  a= 1; b= 2; c= 3; d= 4; e= 5;
  printint(a); printint(b); printint(c); printint(d); printint(e);
  return(0);
}
```

我们编写一个单独的文件`input27b.c`，并用`gcc`编译它：

```c
#include <stdio.h>
extern int param8(int a, int b, int c, int d, int e, int f, int g, int h);
extern int param5(int a, int b, int c, int d, int e);
extern int param2(int a, int b);
extern int param0();

int main() {
  param8(1,2,3,4,5,6,7,8); puts("--");
  param5(1,2,3,4,5); puts("--");
  param2(1,2); puts("--");
  param0();
  return(0);
}
```

然后我们可以将它们链接在一起，看看可执行文件是否运行：

```
cc -o comp1 -g -Wall cg.c decl.c expr.c gen.c main.c misc.c
      scan.c stmt.c sym.c tree.c types.c
./comp1 input27a.c
cc -o out input27b.c out.s lib/printint.c 
./out
1
2
3
4
5
6
7
8
--
1
2
3
4
5
--
1
2
3
4
5
--
1
2
3
4
5
```

而且很管用！我在里面加了一个感叹号，因为当事情成功的时候，有时感觉还是很神奇。让我们检查`param8()`的汇编代码：

```assembly
param8:
        pushq   %rbp                    # Save %rbp, move %rsp
        movq    %rsp, %rbp
        movl    %edi, -4(%rbp)          # Copy six arguments into locals
        movl    %esi, -8(%rbp)          # on the stack
        movl    %edx, -12(%rbp)
        movl    %ecx, -16(%rbp)
        movl    %r8d, -20(%rbp)
        movl    %r9d, -24(%rbp)
        addq    $-32,%rsp               # Lower stack pointer by 32
        movslq  -4(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print -4(%rbp), i.e. a
        movq    %rax, %r11
        movslq  -8(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print -8(%rbp), i.e. b
        movq    %rax, %r11
        movslq  -12(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print -12(%rbp), i.e. c
        movq    %rax, %r11
        movslq  -16(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print -16(%rbp), i.e. d
        movq    %rax, %r11
        movslq  -20(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print -20(%rbp), i.e. e
        movq    %rax, %r11
        movslq  -24(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print -24(%rbp), i.e. f
        movq    %rax, %r11
        movslq  16(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print 16(%rbp), i.e. g
        movq    %rax, %r11
        movslq  24(%rbp), %r10
        movq    %r10, %rdi
        call    printint                # Print 24(%rbp), i.e. h
        movq    %rax, %r11
        movq    $0, %r10
        movl    %r10d, %eax
        jmp     L1
L1:
        addq    $32,%rsp                # Raise stack pointer by 32
        popq    %rbp                    # Restore %rbp and return
        ret
```

`input27a.c`中的其他一些函数既有参数变量又有本地声明的变量，所以看起来生成的前导是正确的（好的，足够通过这些测试了！）

## 总结与展望

我尝试了几次才把这个弄对。第一次，我在错误的方向上遍历了局部符号列表，得到了不正确的参数顺序。我误读了 Eli Bendersky 文章中的图像，这导致我的序言在旧的基础指针上错踩。在某种程度上，这很好，因为重写的代码比原始代码要干净得多。

在我们编译器编写旅程的下一部分，我将修改编译器，用任意数量的参数进行函数调用。然后我可以将`input27a.c`和`input27b.c`移动到`tests/`目录中。